# 3_14 在 Core 建立 Entity 與 DTO

> **一句話**:`Entity` 是資料庫那張表的 C# 形狀,`DTO` 是 API 對外收發的資料形狀,兩者分開,內部改動就不會直接影響 API 合約。

## 1. 為什麼需要
- Entity 放資料庫欄位(包含密碼等不該外露的東西)
- DTO 只放該收、該回的欄位,client 不能亂指定內部欄位

## 2. 步驟

```bash
cd eCommerce.Core
mkdir Entities DTO
touch Entities/ApplicationUser.cs
touch DTO/LoginRequest.cs DTO/RegisterRequest.cs DTO/AuthenticationResponse.cs DTO/GenderOptions.cs
```

| 檔案 | 用途 |
|---|---|
| `Entities/ApplicationUser.cs` | 使用者 Entity |
| `DTO/LoginRequest.cs` | 登入請求(Email、Password) |
| `DTO/RegisterRequest.cs` | 註冊請求(Email、Password、PersonName、Gender) |
| `DTO/GenderOptions.cs` | 性別 enum:`Male`、`Female`、`Other` |
| `DTO/AuthenticationResponse.cs` | 登入 / 註冊成功後回傳(UserID、Email、PersonName、Gender、Token、Success) |

```csharp
namespace eCommerce.Core.DTO;

public class RegisterRequest
{
    public string Email { get; set; } = string.Empty;
    public string Password { get; set; } = string.Empty;
    public string PersonName { get; set; } = string.Empty;
    public string Gender { get; set; } = string.Empty;
}
```

> 後來 `AuthenticationResponse` 改成 `record`(見 8_19_20_AutoMapper)。

## 3. 關鍵概念

> **Entity vs DTO**
> Entity 對應資料表(`ApplicationUser`);DTO 是 API 與外界傳資料用(`RegisterRequest`)。

> **Core 層放什麼**
> Entities、DTOs、Service / Repository Contracts(介面)、Services、Mappers。**不依賴** API 與 Infrastructure。

延伸閱讀:[Why API references Core and Infrastructure.md](Why%20API%20references%20Core%20and%20Infrastructure.md)

## 4. 下一步
建立 Repository 介面(Core)與實作(Infrastructure)→ 見 4_15。
