# 1_12 建立 Users 微服務專案結構

> **一句話**:建一個 Solution + 三個專案(API / Infrastructure / Core),用「專案參考」串起依賴方向,各層的 DI 打包成擴充方法,`Program.cs` 只負責呼叫。

## 1. 為什麼需要
先把分層與依賴方向定好,之後每個類別該放哪一層就不會亂。

```
eCommerce.API  ──→  eCommerce.Infrastructure  ──→  eCommerce.Core
      └────────────────────────────────────────────→┘
```

## 2. 步驟

```bash
mkdir eCommerceSolution.UserService && cd eCommerceSolution.UserService
dotnet new sln -n eCommerceSolution.UsersService
dotnet new webapi   -n eCommerce.API
dotnet new classlib -n eCommerce.Infrastructure
dotnet new classlib -n eCommerce.Core
dotnet sln add eCommerce.API eCommerce.Infrastructure eCommerce.Core

# 專案參考
dotnet add eCommerce.API reference eCommerce.Infrastructure
dotnet add eCommerce.Infrastructure reference eCommerce.Core
dotnet add eCommerce.API reference eCommerce.Core

# 檢查
dotnet list eCommerce.API/eCommerce.API.csproj reference
```

另外在 `eCommerce.API` 放一個 `Dockerfile`(之後 Docker 章節使用)。

**每層的 `DependencyInjection.cs`**(Infrastructure、Core 各一個)是該層對外的「服務選單」:

```csharp
// Program.cs 只需要這兩行,不用知道資料庫、商業邏輯怎麼寫
builder.Services.AddInfrastructure();
builder.Services.AddCore();
```

## 3. 關鍵概念

> **classlib(Class Library)**
> 編譯成 `.dll` 的類別庫,沒有 `Program.cs`,不能獨立執行。`webapi` 才是進入點。

> **能 `using` 別的專案,要兩個條件缺一不可**
> 1. **Reference**:搭橋,編譯器才「看得到」那個專案
> 2. **`public`**:開門,`internal` 或沒寫的類別,外人不能用

> **`this IServiceCollection` 擴充方法**
> 讓寫在 Infrastructure 的 `AddInfrastructure()` 看起來像 `builder.Services` 的內建功能。

### Program.cs 的三個階段
| 階段 | 做什麼 |
|---|---|
| ① `CreateBuilder` | 建立 builder,預設設好 Kestrel、`appsettings.json`、Logging |
| ② 登記服務 | `AddInfrastructure()`、`AddCore()`、`AddControllers()` 放進 IoC 容器 |
| ③ 中間件管線 | `Build()` 之後的 `app.Use...`,請求依序通過 |

### 中間件管線(請求由上往下通過)
| 順序 | 方法 | 作用 |
|---|---|---|
| 1 | `UseRouting()` | 找網址對應哪個 Controller |
| 2 | `UseAuthentication()` | 你是誰?(解析 JWT / Cookie) |
| 3 | `UseAuthorization()` | 你能做什麼? |
| 4 | `MapControllers()` | 放行,交給 Controller |
| 最後 | `app.Run()` | 開始監聽 Port |

> ⚠️ `UseAuthentication()` 一定要在 `UseAuthorization()` 前面:先知道你是誰,才能決定你能不能做。
