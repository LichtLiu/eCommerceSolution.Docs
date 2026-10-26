# 4_15 Repository:Core 放介面,Infrastructure 放實作

> **一句話**:Core 定義「我需要一個能存取使用者的工具」(`IUsersRepository`),Infrastructure 負責實作(`UsersRepository`)。商業邏輯不認識資料庫,這就是**依賴反轉**。

## 1. 為什麼需要
Service 不該直接碰資料庫。把「怎麼存取」抽成介面,Core 就不綁死任何資料庫技術,測試時也能換成假的。

## 2. 步驟

**① Core:`RepositoryContracts/IUsersRepository.cs`**(介面,`I` 開頭)

```csharp
using eCommerce.Core.Entities;
namespace eCommerce.Core.RepositoryContracts;

public interface IUsersRepository
{
    Task<ApplicationUser?> AddUser(ApplicationUser user);
    Task<ApplicationUser?> GetUserByEmail(string? email);
}
```

**② Infrastructure:`Repositories/UsersRepository.cs`**(實作介面)

```csharp
using eCommerce.Core.RepositoryContracts;
namespace eCommerce.Infrastructure.Repositories;

public class UsersRepository : IUsersRepository
{
    // 之後在這裡用 EF Core 的 DbContext 實作(見 9_PostgreSQL_EFCore)
}
```

**③ Infrastructure 的 `DependencyInjection.cs` 註冊**

```csharp
services.AddScoped<IUsersRepository, UsersRepository>();
```

```bash
dotnet build   # Build succeeded 代表三步都接通
```

## 3. 關鍵概念

> **Contract(合約)= 介面**
> 只規定「要做什麼」,不寫「怎麼做」。

> **依賴反轉原則(Dependency Inversion)**
> 高階的商業邏輯(Core)不依賴低階的資料庫技術(Infrastructure);反而是 Infrastructure 去實作 Core 定義的介面。依賴方向永遠指向 Core。

> **`AddXxx<介面, 實作>()` 的意思**
> 「以後有人在建構子要這個介面,就幫他 new 一個實作塞進去。」

| 名詞 | 說明 |
|---|---|
| `IUsersRepository` | Core 層,抽象 |
| `UsersRepository` | Infrastructure 層,具體實作 |
| `: IUsersRepository` | 宣告實作該介面 |

> 一開始講師用 `AddSingleton`(因為還沒接真的資料庫,資料暫存在記憶體)。接上 EF Core 後**必須改成 `AddScoped`**,原因見 [Difference between Singleton and Scoped.md](Difference%20between%20Singleton%20and%20Scoped.md)。

## 4. 下一步
建立 Service 的合約與實作 → 見 5_16。
