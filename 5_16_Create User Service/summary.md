# 5_16 Service:Core 的商業邏輯

> **一句話**:Service 只管商業規則(查有沒有這個人、密碼對不對),「怎麼存取資料庫」全部交給 Repository。

## 1. 為什麼需要
Controller 只負責收發 HTTP,商業邏輯集中在 Service,之後換 UI 或資料庫都不影響。

## 2. 步驟

**① 合約:`Core/ServiceContracts/IUsersService.cs`**

```csharp
namespace eCommerce.Core.ServiceContracts;

public interface IUsersService
{
    Task<AuthenticationResponse?> Login(LoginRequest request);
    Task<AuthenticationResponse?> Register(RegisterRequest request);
}
```

**② 實作:`Core/Services/UsersService.cs`**(注入 `IUsersRepository`)

| 方法 | 流程 |
|---|---|
| `Login` | 拿 Email / Password 叫 Repository 查 → 找不到回 `null` → 找到就轉成 `AuthenticationResponse` |
| `Register` | `RegisterRequest` 轉成 `ApplicationUser` → `AddUser` 存入 → 轉成 `AuthenticationResponse` 回傳 |

**③ 註冊:Core 的 `DependencyInjection.cs`**

```csharp
services.AddTransient<IUsersService, UsersService>();
```

## 3. 關鍵概念

> **為什麼 DI 註冊 `<介面, 實作>` 而不是直接 `new`**
> - **解耦**:Controller 只依賴介面,不知道背後怎麼做
> - **可替換**:改成 Google 登入只要寫 `GoogleUsersService`,改一行註冊
> - **可測試**:單元測試塞假的 Mock,不必動真實資料庫

> **Service 為什麼用 Transient**
> Service 無狀態(不存資料),每次新建最安全。

> ⚠️ 當時 Login / Register 的密碼是**明碼**直接比對與存入。後來在 9_PostgreSQL_EFCore 改成 BCrypt 雜湊。

## 4. 下一步
建立 `AuthController` → 見 6_17。
