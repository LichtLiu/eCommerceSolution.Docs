# 9 接上 PostgreSQL(Docker + EF Core)

> **一句話**:Repository 從「假資料」變成真的讀寫 PostgreSQL;資料庫跑在 Docker,ORM 用 EF Core,密碼用 BCrypt 雜湊。

## 1. 為什麼需要
之前 `UsersRepository` 是假的:`AddUser` 只產生 Guid,`GetUserByEmailAndPassword` 永遠回同一個使用者。

## 2. 步驟

**① Docker 跑 PostgreSQL**
```bash
docker run -d --name ecommerce-postgres \
  -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=<密碼> -e POSTGRES_DB=ecommerce_users \
  -p 5432:5432 \
  -v ecommerce_pgdata:/var/lib/postgresql/data \
  postgres:17
```
驗證:`docker ps` 看到 `Up`。

**② 套件**(版本要一致,否則 `NU1605` downgrade 錯誤)
- Infrastructure:`Microsoft.EntityFrameworkCore`、`Npgsql.EntityFrameworkCore.PostgreSQL`、`Microsoft.Extensions.Configuration.Abstractions`(10.0.13)
- API:`Microsoft.EntityFrameworkCore.Design`

**③ `Infrastructure/Data/ApplicationDbContext.cs`**
```csharp
public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options) : base(options) { }

    public DbSet<ApplicationUser> Users { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        modelBuilder.Entity<ApplicationUser>(entity =>
        {
            entity.HasKey(u => u.UserID);               // 主鍵
            entity.HasIndex(u => u.Email).IsUnique();   // Email 不可重複
        });
    }
}
```

**④ 連線字串**(API 的 `appsettings.json`)
```json
"ConnectionStrings": {
  "DefaultConnection": "Host=localhost;Port=5432;Database=ecommerce_users;Username=postgres;Password=<密碼>"
}
```

**⑤ 註冊**(`Infrastructure/DependencyInjection.cs`,`Program.cs` 傳入 `builder.Configuration`)
```csharp
public static IServiceCollection AddInfrastructure(this IServiceCollection services, IConfiguration configuration)
{
    services.AddDbContext<ApplicationDbContext>(options =>
        options.UseNpgsql(configuration.GetConnectionString("DefaultConnection")));
    services.AddScoped<IUsersRepository, UsersRepository>();
    return services;
}
```

**⑥ Migration**(在**專案根目錄**執行)
```bash
dotnet ef migrations add InitialCreate --project eCommerce.Infrastructure --startup-project eCommerce.API
dotnet ef database update            --project eCommerce.Infrastructure --startup-project eCommerce.API
```

**⑦ Repository 真的存取**
```csharp
public async Task<ApplicationUser> AddUser(ApplicationUser user)
{
    user.UserID = Guid.NewGuid();
    _dbContext.Users.Add(user);           // 只是標記
    await _dbContext.SaveChangesAsync();  // 這行才真的 INSERT
    return user;
}

public async Task<ApplicationUser?> GetUserByEmail(string? email)
    => await _dbContext.Users.FirstOrDefaultAsync(u => u.Email == email);
```

## 3. 關鍵概念

> **DbContext 與 Repository 用 Scoped**
> `DbContext` 不是 thread-safe、會追蹤狀態,每個請求一個。Repository 依賴它,生命週期要一致。

> **一個 DbContext 同一時間只能做一件事**
> 同一請求內不要用 `Task.WhenAll` 同時跑多個查詢。

| 概念 | 說明 |
|---|---|
| 表名來自 `DbSet` 屬性名 | `DbSet<ApplicationUser> Users` → 表叫 `Users`,想固定用 `ToTable("Users")` |
| 主鍵慣例 | EF 只認 `Id` 或 `類別名Id`;`UserID` 要用 `HasKey` 指定(不在 Entity 加 `[Key]`,避免 Core 被資料庫細節污染) |
| Guid vs int 主鍵 | Guid 由程式產生(存之前就有);int 由資料庫產生(存之後才知道) |
| `--project` vs `--startup-project` | project:Migration 放哪(Infrastructure);startup-project:用誰的設定跑(API,有 appsettings) |
| `SaveChangesAsync()` | 沒呼叫就不會寫入 |
| 建構函式名稱 | 必須跟**類別名**一樣,不是檔名 |

> **Service 在 Core、Repository 在 Infrastructure**
> Service 只依賴 interface,不碰外部技術;Repository 用到 EF Core 所以放外層。依賴方向永遠指向 Core。

### 密碼雜湊(BCrypt.Net-Next,裝在 Core)
| | 做法 |
|---|---|
| Register | `BCrypt.Net.BCrypt.HashPassword(密碼)` 再存,資料庫看到 `$2a$11$...` |
| Login | 不能拿雜湊去查。先 `GetUserByEmail` 找人,再 `BCrypt.Net.BCrypt.Verify(輸入, 資料庫雜湊)` |
| 重複註冊 | `AddUser` 先 `AnyAsync` 檢查 Email,存在回 `null` → Controller 回 400;unique index 保留當最後防線(兩人同時註冊只有資料庫能擋) |

> 找不到 Email 與密碼錯誤都回 `null`,不讓對方分辨是哪個錯(避免探測哪些 Email 存在)。BCrypt 每次自動加隨機 salt,同密碼每次結果不同,所以一定要用 `Verify`。

## 常見錯誤
| 症狀 | 原因 / 解法 |
|---|---|
| `dotnet ef` 路徑錯誤 | 在專案根目錄執行,路徑是相對的 |
| `database update` 出現 `fail: SELECT ... __EFMigrationsHistory` | 第一次執行的正常現象,看最後有沒有 `Done.` |
| `requires a primary key` | 主鍵名稱不符慣例,用 `HasKey` |
| 重複註冊回 500 | unique index 讓 `SaveChangesAsync` 丟 `DbUpdateException`,沒人接,被 Middleware 變 500 |

## 常用指令
```bash
docker exec -it ecommerce-postgres psql -U postgres -d ecommerce_users -c '\dt'
docker exec -it ecommerce-postgres psql -U postgres -d ecommerce_users -c '\d "Users"'
docker exec -it ecommerce-postgres psql -U postgres -d ecommerce_users -c 'SELECT * FROM "Users";'
```
> PostgreSQL 對大小寫敏感,表名要加雙引號 `"Users"`。

## 5. 測試結果與下一步
- Register 200、資料進 `Users`;Login 200,`userID` 與 Register 相同(確認從資料庫查出)。
- 已知問題:Email 沒有格式驗證(下一章解決)、Token 仍是寫死的 `"token"`(JWT 待做)。
