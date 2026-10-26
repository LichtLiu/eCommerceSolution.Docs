# 8_19_20 AutoMapper

> **一句話**:把「一個物件的屬性複製到另一個物件」的規則集中在 `Profile`,用 `_mapper.Map<目標>(來源)` 一行搞定。

## 1. 為什麼需要
UserService 裡有很多手動 `new AuthenticationResponse` 再逐一抄屬性的程式碼。規則散落各處,改一次要改很多地方。AutoMapper 集中管理,改 Profile 全專案同步生效。

## 2. 步驟

**① 套件**:Core 專案安裝 `AutoMapper`(API 專案也要裝,兩邊**版本相同**)。

**② `Core/Mappers/ApplicationUserMappingProfile.cs`**
```csharp
public class ApplicationUserMappingProfile : Profile
{
    public ApplicationUserMappingProfile()   // 規則寫在建構函式
    {
        CreateMap<ApplicationUser, AuthenticationResponse>()
            .ForMember(dest => dest.UserID,     opt => opt.MapFrom(src => src.UserID))
            .ForMember(dest => dest.Email,      opt => opt.MapFrom(src => src.Email))
            .ForMember(dest => dest.PersonName, opt => opt.MapFrom(src => src.PersonName))
            .ForMember(dest => dest.Gender,     opt => opt.MapFrom(src => src.Gender))
            .ForMember(dest => dest.Success,    opt => opt.Ignore())
            .ForMember(dest => dest.Token,      opt => opt.Ignore());
    }
}
```

**③ `Program.cs` 註冊**(AutoMapper 16 的寫法)
```csharp
builder.Services.AddAutoMapper(cfg => cfg.AddMaps(typeof(ApplicationUserMappingProfile).Assembly));
```
> 舊版寫法 `AddAutoMapper(typeof(...))` 在 16.2.0 會編譯失敗(`CS1503`)。這行同時註冊 `IMapper`,且會掃描整個 assembly 的所有 `Profile`。

**④ Service 使用**(注入 `IMapper`)
```csharp
return _mapper.Map<AuthenticationResponse>(user) with { Success = true, Token = "token" };
```

**⑤ 修「需要無參數建構函式」錯誤**:`AuthenticationResponse` 是 record,AutoMapper 要先建空物件,所以加無參數建構函式:
```csharp
public record AuthenticationResponse(
    Guid UserID, string? Email, string? PersonName, string? Gender, string? Token, bool Success)
{
    public AuthenticationResponse() : this(default, default, default, default, default, default) { }
}
```
> 主要建構函式的 `)` 後面**不能加分號**,否則 record 提前結束,後面的 `{...}` 變成語法錯誤。

## 3. 關鍵概念

| 方法 | 用途 |
|---|---|
| `CreateMap<來源, 目標>()` | 宣告單向對應(反向要另外宣告) |
| `ForMember` | 指定**目標**屬性 |
| `MapFrom` | 指定**來源**屬性 |
| `Ignore` | 目標屬性不指定值,維持預設 |

> **`with` 運算式(C# 9)**
> 只能用在 record / struct。複製物件並換掉指定屬性,回傳**新物件**,原物件不變。這裡用來補上被 `Ignore()` 的 `Token`、`Success`。

> **Profile 要自己寫**
> 套件提供對應功能,但「哪個來源屬性對哪個目標屬性」不會自動產生,要寫在 Profile。

> ⚠️ AutoMapper 15 之後是商業授權,16.x 執行時可能出現缺少 license key 的警告,學習用可忽略。

## 4. 作業
在同一個 `Mappers` 資料夾新增另一個 Profile,`RegisterRequest` → `ApplicationUser`,取代 `Register` 裡手動複製的程式碼。

## 5. 測試結果與下一步
- 套用後 login 一度報「需要無參數建構函式」,依上面 ⑤ 修正。
- 下一步:接真正的資料庫 → 見 9_PostgreSQL_EFCore。
