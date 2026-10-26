# 12 Products 微服務:建立專案與 DAL

> **一句話**:作業要做 Products 微服務,分 DAL / BLL / API 三層;先做最底層 DAL(Entity、DbContext、Repository、Migration),讓上層只依賴介面。

## 1. 為什麼需要

| 作業的層 | 對應 UserService | 內容 |
|---|---|---|
| DataAccessLayer(DAL) | Infrastructure | Entity、DbContext、Repository、DI |
| BusinessLogicLayer(BLL) | Core | DTO、Service、Validator、Mapper |
| APILayer | API | Minimal API、Middleware、`Program.cs` |

依賴方向:`APILayer → BLL → DAL`(專案參考預設會**傳遞**,API 技術上不用直接參考 DAL)。

## 2. 步驟
1. **建 Solution**:`dotnet new classlib` ×2(DAL、BLL)+ `dotnet new webapi`(API),`dotnet sln add`,`dotnet add reference`
2. **`Product` Entity**(`[Key] Guid ProductID`,其餘欄位 nullable)+ **`ApplicationDbContext`**(`DbSet<Product>`)
3. **`IProductsRepository` / `ProductsRepository`**(CRUD,條件用 `Expression<Func<Product,bool>>`)
4. **`DependencyInjection.AddDataAccessLayer`**:`AddDbContext` + `UseMySQL` + `AddScoped`
5. **Migration**,用 `DESCRIBE Products` 驗證

```bash
dotnet ef migrations add InitialCreate --project DataAccessLayer --startup-project APILayer --output-dir Migrations
dotnet ef database update            --project DataAccessLayer --startup-project APILayer
```

## 3. 關鍵概念

> **EF Core 裝在 DAL**
> 用到 EF 的那層才裝;BLL 只認 `IProductsRepository`,不認 EF。

> **`Expression<Func<>>` vs `Func<>`**
> Expression 是「條件的描述」,EF 能翻成 SQL `WHERE`;Func 會先把全表撈進記憶體再過濾。

> **`migrations add` 不連資料庫,`database update` 才連**
> `add` 只比對「目前模型」與 `ModelSnapshot`,產生程式碼。`update` 才執行 SQL 並寫入 `__EFMigrationsHistory`。

| 名詞 | 用途 |
|---|---|
| classlib | 編譯成 dll 的類別庫,不能獨立執行;webapi 才有進入點 |
| MySQL 套件 | Pomelo 當時無 EF 10 穩定版 → 用官方 `MySql.EntityFrameworkCore`,方法是 `UseMySQL`(SQL 大寫) |
| `Scoped` vs `Singleton` | DbContext 非 thread-safe,Singleton 會並行衝突;Scoped = 每請求一個 |
| `--startup-project` | `ef` 要執行這個專案的 `Program.cs` 才拿得到 DbContext 設定 |
| `Unknown column` / `Table doesn't exist` | 模型與資料庫不同步,先 `migrations list` 看有沒有 Pending |

### 踩過的坑
- csproj 不見 → 重建並重新 `dotnet add package`(之後用 git 存檔預防)
- Interface 檔 `using` 不對(IDE 自動產生 `using System;`)→ 補 `System.Linq.Expressions`、`DataAccessLayer.Entities`
- Entity 主鍵寫成 `Id`,作業要求 `ProductID`
- `DependencyInjection.cs` 沒建 → `AddDataAccessLayer` 找不到
- `Program.cs` 沒有 `Build()` / `Run()` → `dotnet ef` 報 `entry point exited without ever building an IHost`

## 4. 作業(自己練習)
1. `ProductName` 加 `[Required]` 與 `[MaxLength(100)]`,做新 Migration,觀察 `DESCRIBE Products`
2. `Guid.NewGuid()` 放 Repository / Service / Mapper 的利弊,看老師答案放哪
3. 把 `AddScoped` 暫時改成 `AddSingleton`,看啟動時的錯誤訊息

## 5. 結果與下一步
`dotnet build` 成功;MySQL `Products` 表與 Entity 一致。下一步:BLL(見 13)。
