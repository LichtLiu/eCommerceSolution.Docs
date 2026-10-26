# 10 輸入驗證(FluentValidation)

> **一句話**:在資料進 Service 之前就擋掉不合法輸入,回 `400` 並告訴使用者哪裡錯。

## 1. 為什麼需要
沒驗證時,`123@mail`、空密碼、空名字都能註冊成功。

## 2. 設計
- Validator 放 **Core/Validators**(業務規則,不碰外部技術)
- 在 **Controller** 呼叫 `ValidateAsync`,失敗回 `BadRequest` 加錯誤清單
- **Register** 檢查格式與強度;**Login 只檢查有沒有填**(不洩漏密碼規則,也不擋舊帳號)

## 3. 步驟
1. Core 安裝 `FluentValidation`、`FluentValidation.DependencyInjectionExtensions`(12.0.0)
2. 新增 `RegisterRequestValidator`、`LoginRequestValidator`,規則寫在**建構函式**裡
3. Core 的 `DependencyInjection`:
   ```csharp
   services.AddValidatorsFromAssemblyContaining<RegisterRequestValidator>();   // 記得 using eCommerce.Core.Validators;
   ```
4. Controller 建構子注入 `IValidator<RegisterRequest>`、`IValidator<LoginRequest>`,action 一開始先驗證

```csharp
RuleFor(x => x.Email)
    .NotEmpty().WithMessage("Email 不可為空")
    .Matches(@"^[^@\s]+@[^@\s]+\.[^@\s]+$").WithMessage("Email 格式不正確");
RuleFor(x => x.Password)
    .NotEmpty().WithMessage("密碼不可為空")
    .MinimumLength(8).WithMessage("密碼至少 8 個字元");
```

## 4. 關鍵概念

> **`AddValidatorsFromAssemblyContaining<T>()`**
> 型別只是定位點,會掃描 T 所在整個 assembly,註冊所有 `AbstractValidator<>`。跟 AutoMapper 掃描 Profile 同概念。

| 概念 | 說明 |
|---|---|
| `EmailAddress()` | 內建規則很寬鬆,放行 `123@mail`;要求網域尾巴就用 `Matches` regex |
| 規則要寫在建構函式 | class 裡不能直接放敘述句(`RuleFor(...)`),否則冒出一大串 `',' expected` |
| 改程式碼後要重啟 API | 沒重啟會跑到舊版本;可用 `dotnet watch run` |

## 常見錯誤
- 一大串 `',' expected` / `Invalid token in a member declaration` → 看**第一個**錯誤,通常是敘述句放在 class 層級,或括號沒對上
- `CS0246 type could not be found` → 少了 `using`
- interface 與實作的 `?`(nullable)不一致 → CS8613 / CS8604 警告,兩邊簽章要一致

## 5. 測試結果與下一步
- `123@mail` + `12345678` → 400,`Email 格式不正確`
- `123@mail` + `123` → 400(密碼太短)
- 合法輸入 → 200
- 下一階段:JWT(Token 目前仍是寫死的 `"token"`)
