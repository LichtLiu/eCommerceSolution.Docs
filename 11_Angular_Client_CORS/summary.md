# 11 Angular Client 接上 UserService(含 CORS)

> **一句話**:Postman 不是瀏覽器所以沒擋過跨來源請求;真的用瀏覽器(Angular)呼叫 API 時,後端要允許那個來源,這就是 CORS。

## 1. 為什麼需要
老師給了 Angular UI(登入、註冊)。接上它能在真實瀏覽器驗證 API,也第一次碰到 CORS。

## 2. 步驟
1. 把老師的 `19. Angular Client App` 複製到自己資料夾 `eCommerceSolution.AngularClient`(不要改老師的 repo),`npm install`
2. 改 `src/environment.ts` 的 `apiUrl`:`7186` → 自己 API 的 `https://localhost:7032/api/auth/`
3. UserService `Program.cs` 加 CORS:
   ```csharp
   builder.Services.AddCors(options =>
   {
       options.AddDefaultPolicy(policy =>
           policy.WithOrigins("http://localhost:4200").AllowAnyHeader().AllowAnyMethod());
   });

   // 順序:UseRouting 之後、UseAuthentication 之前
   app.UseCors();
   ```
4. `dotnet dev-certs https --trust`(讓瀏覽器信任本機 HTTPS 憑證)
5. 兩個終端機:`dotnet run --project eCommerce.API --launch-profile https` 與 `npm start`,開 `http://localhost:4200`

## 3. 關鍵概念

> **來源(Origin)= 協定 + 網域 + port**
> `localhost:4200` 和 `localhost:7032` 是不同來源。三者全部相同才算同源。

> **CORS 是瀏覽器執行的規則**
> 跨來源請求,後端要回 `Access-Control-Allow-Origin` 才放行。Postman / curl 不受限。

> **CORS 不是安全防護**
> 它保護使用者的瀏覽器,不能取代登入驗證(JWT)。

| 概念 | 說明 |
|---|---|
| `AllowAnyOrigin()` 的風險 | 任何網站的 JS 都能從使用者瀏覽器呼叫並讀取你的 API 回應。正式環境用 `WithOrigins(...)`,網址放設定檔 |
| `UseCors()` 的位置 | `UseRouting` 之後、`UseAuthentication` / `UseAuthorization` 之前 |
| 401 vs CORS 被擋 | 401:請求到達 API,登入失敗,`status = 401`。CORS 被擋:`status = 0`,Console 顯示 `blocked by CORS policy` |
| Angular 前後端是兩個程序 | API(7032)與 `ng serve`(4200)各跑各的 |
| `HttpClient` + `Observable` | `this.http.post<T>(url, body)` 回傳 Observable,要 `.subscribe({ next, error })` 才真的送出 |
| `localStorage` 存 token | `UsersService` 登入後把 token 與使用者名稱存進去 |

## 4. 前後端對不上的地方(實際遇到)
| 項目 | UI | API | 處理 |
|---|---|---|---|
| API 網址 | `7186` | `7032` | 改 `environment.ts` |
| Gender | `Male/Female/Others` | enum `Male/Female/Other` | 選 `Others` 會 400,待修 |
| 密碼長度 | 最短 3 | 最短 8 | `123` 被後端擋,UI 只有 `console.log`,使用者看不到原因 |

## 常見錯誤
- 登入 401 → 先確認帳號存在、密碼正確(資料庫只存 BCrypt 雜湊,無法反查密碼)
- 改了 API 沒生效 → 要重啟 API
- 瀏覽器看不到 API 回應但 Postman 正常 → 多半是 CORS

## 5. 測試結果與下一步
- 註冊、登入成功;密碼 `123` 被擋;CORS 正常。
- 下一步:Products 微服務(見 12、13);之後回來讀懂 Angular,並修 `Others` 與錯誤顯示。
