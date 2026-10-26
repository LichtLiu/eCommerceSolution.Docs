# 13 Products 微服務:BLL、API、Swagger、CORS、交作業

> **一句話**:DTO 隔離對外資料形狀、Service 放商業邏輯、Validator 擋壞資料;Minimal API 把 Service 變成 HTTP、Middleware 統一處理例外;最後接上 Angular。

## 1. 為什麼需要
- BLL:DTO、Service、Validator
- API:Minimal API、例外 Middleware
- Bonus:Swagger 自動產生文件、CORS 讓 Angular 能呼叫

依賴方向:`APILayer → BLL → DAL`。

## 2. 步驟

| # | 做什麼 | 重點 |
|---|---|---|
| 5 | **DTO**(`record`) | `ProductResponse`、`ProductAddRequest`(**沒有** ProductID)、`ProductUpdateRequest`(有 ProductID) |
| 6 | **AutoMapper** | `ProductMappingProfile`;Add → Entity 要 `.ForMember(ProductID, Ignore())` |
| 7 | **Service** | `IProductsService` / `ProductsService`(注入 `IProductsRepository` + `IMapper`),`AddScoped` |
| 8 | **FluentValidation** | `ProductAddRequestValidator`、`ProductUpdateRequestValidator`,`AddValidatorsFromAssemblyContaining<T>()` |
| 9 | **Minimal API** | `MapProductAPIEndpoints()`,`MapGroup("/api/products")`;搜尋路徑為 `/api/products/search/{searchString}`(作業文字的 `/products/search/` 是筆誤,見 13b) |
| 10 | **API 接 Validator** | endpoint 注入 `IValidator<T>`,`ValidateAsync`,不合法回 `Results.ValidationProblem(...)`(400) |
| 11 | **例外 Middleware** | `try/catch` 包 `await _next(ctx)`,log + 固定 500,`UseExceptionHandlingMiddleware()` 放最前面 |
| 12 | **Swagger + CORS** | `AddEndpointsApiExplorer` / `AddSwaggerGen` / `UseSwagger` / `UseSwaggerUI`;`WithOrigins("http://localhost:4200")` + `UseCors()` 在 endpoint 之前 |
| 13 | **交作業** | README、push GitHub(Products、AngularClient 各一個 repo) |

```csharp
// endpoint 內驗證
ValidationResult result = await validator.ValidateAsync(request);
if (!result.IsValid)
    return Results.ValidationProblem(ToErrorDictionary(result));
```

### Endpoints
| 方法 | 路徑 | 成功 | 錯誤 |
|---|---|---|---|
| GET | `/api/products` | 200 | |
| GET | `/api/products/search/product-id/{productID}` | 200 | 404 |
| GET | `/api/products/search/{searchString}` | 200 | |
| POST | `/api/products` | 201 | 400 |
| PUT | `/api/products` | 200 | 400, 404 |
| DELETE | `/api/products/{productID}` | 200 | 404 |

## 3. 關鍵概念

> **DTO 與 mass assignment**
> client 傳來的資料一律不可信,內部欄位(ProductID)只由伺服器決定。Add 沒有 ID,Update 才帶 ID(是「改哪筆」的條件,不是新值)。

> **`AssertConfigurationIsValid()`**
> AutoMapper 啟動時檢查有沒有漏對應,把「故意」(`Ignore`)和「疏忽」分開。

> **Middleware 順序 = 執行順序**
> 例外 Middleware 放最外層,才接得到後面所有環節的例外。**不回 `ex.Message`**,只寫 log。

> **`NotEmpty()` vs `NotNull()`**
> 數字的 `NotEmpty()` 會擋掉 `0`;必填但 0 合法要用 `NotNull()`。Guid 的 `NotEmpty()` 剛好擋 `Guid.Empty`。

| 名詞 | 用途 |
|---|---|
| 依賴介面注入 | 單元測試可 Mock Repository(Mapper 用真的),不必連資料庫 |
| `Add` / `Use` / `Map` | 註冊服務 / 加 middleware / 對應路由,命名慣例 |
| GET vs POST | GET 唯讀、參數放網址;POST / PUT 改資料、放 Body;敏感資料不放網址 |
| 專案參考傳遞 | A→B→C 時 A 看得到 C |
| `NU1903` 警告 | 範本的 `Microsoft.AspNetCore.OpenApi` 帶進有漏洞的 `Microsoft.OpenApi`;不用就移除。Swashbuckle 10.3.1 無此警告 |
| CORS | origin = 協定 + 網域 + port,**逐字串**比對;`WithOrigins` 結尾不能有 `/`;只擋瀏覽器,curl / Postman 不受影響 |
| 驗證長度要和資料庫一致 | 驗證 200、DB `varchar(100)`:101~200 字通過驗證卻在 DB 失敗 → 500 而非 400 |
| 直接灌 SQL 繞過驗證 | 資料庫約束是最後防線;FluentValidation 只擋經過 API 的資料 |
| 多 repo vs monorepo | 每服務一個 repo;GitHub Project 只是看板,不是 repo 容器,Organization 才能群組 repo |

### 踩過的坑
- Postman 把 GET 的 ID 查詢寫成 `POST` + Body `{"id":"1"}` → ID 要放網址、方法要 GET、ID 必須是 Guid
- POST `{}` 回 201 並存進全 null 資料 → Validator 寫了但 endpoint 沒呼叫
- 改名只改一半(`AddProductAPIEndpoints` vs `MapProductAPIEndpoints`)→ 定義與呼叫要一致
- `WithOrigins("http://localhost:4200/")` 結尾多斜線會失效

## 4. 作業(自己練習)
1. 兩個欄位加 `[MaxLength(100)]`,做 Migration,看 `DESCRIBE Products`
2. 寫 `ProductsService` 單元測試(Mock `IProductsRepository`,真 AutoMapper):找不到商品回 `null`
3. 比較 `PUT` 把 ID 放 Body 與放網址(`PUT /api/products/{id}`)的利弊
4. 讓 Swagger 只在 `Development` 開啟
5. 連線字串改用 `dotnet user-secrets`;看老師影片對照 Guid 產生位置

## 5. 結果與下一步
Validator 回 400、例外 Middleware 回固定 500、Swagger、CORS 皆實測通過;Products 與 AngularClient 已 push。接 Angular 見 13b。
