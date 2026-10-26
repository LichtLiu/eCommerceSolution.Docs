# 祕密不要打包進 Image

> **一句話**:`appsettings.json` 會被 `publish` 進 Image,連線字串、密碼就成了 layer 的內容。**祕密在執行時才從外面給,不要寫進 Image。**

## 問題:連線字串會被打包進去

```
layer:publish 的編譯結果
  ├ APILayer.dll
  └ appsettings.json   ← 含 Server=...;Password=...
```

任何拿到這個 Image 的人,都能 `docker pull` 之後進去讀這個檔案。

| 點 | 說明 |
|---|---|
| private 只是降低風險 | 被授權的人、或帳號被盜,仍拿得到。祕密放進 Image 本身就是壞習慣 |
| layer 刪不掉 | 後面的指令刪除檔案,**前面 layer 裡的檔案還在**,只是被蓋住(分層檔案系統的特性,我的理解,未實際驗證) |

> **我的答案(Docker Hub 該選 private 還是 public)**:選 private,因為 Image 裡有編譯結果,連線字串也會被打包進去。

## 做法:執行時才注入

| 做法 | 說明 |
|---|---|
| 環境變數 | `docker run -e ConnectionStrings__DefaultConnection="Server=...;Password=..." ...` |
| compose / 平台的 secret | 之後 docker-compose、Kubernetes 的 Secret 都是做這件事 |

> ASP.NET Core 的設定可以被環境變數覆蓋,階層用雙底線 `__`:`ConnectionStrings__DefaultConnection` 會蓋掉 `appsettings.json` 的 `ConnectionStrings:DefaultConnection`。(我的知識,等老師講環境變數時核對。)
> 已用過同類寫法:`docker run -e MYSQL_ROOT_PASSWORD=...`。

這樣 Image 裡只留**不含祕密的預設值**,就算 Image 是 public 也不會洩漏。

## 連回我的專案

- Products repo 的 `appsettings.json` 有 `Password=mysql123`,已在公開的 GitHub 上。本機測試密碼,風險小,但 Docker 化時要改掉這個習慣。
- UserService 的 `appsettings.json` 也有 PostgreSQL 連線字串,同樣處理。

## 問題:`DefaultConnection` 刪掉後,本機 `dotnet run` 會怎樣?

> **我的答案**:沒有連線字串的參數,就會跳錯誤。

**實測**(在 Products 專案的暫存副本,拿掉 `ConnectionStrings` 整段):

| 時間點 | 結果 |
|---|---|
| 啟動 | **正常**,Swagger 回 200,不會當場報錯 |
| 第一個要用資料庫的請求(`GET /api/products`) | `500`,body 是固定的 `{"error":"An unexpected error occurred..."}` |
| log | `ArgumentNullException: Value cannot be null. (Parameter 'connectionString')`,來自 `UseMySQL(...)` |

> **為什麼不是啟動就失敗**:`AddDbContext(options => options.UseMySQL(...))` 的 lambda 是**延遲執行**,第一次需要建立 `DbContext` 時才跑。DI 裡寫的 `GetConnectionString(...)!`,最後的 `!` 只是消掉編譯器警告,**不會檢查值**,null 一路傳到 `UseMySQL` 才爆。

**連回 Docker**:連線字串改用環境變數,若 `docker run` 忘了帶 `-e`,現象是「容器啟動成功、看起來正常,第一個請求才 500」。排查:`docker logs <容器>`,找 `ArgumentNullException ... connectionString`。

**Fail fast(進階、可選,作業沒要求)**:啟動時就檢查,讀不到就丟明確訊息:

```csharp
var cs = configuration.GetConnectionString("DefaultConnection")
    ?? throw new InvalidOperationException("Missing ConnectionStrings:DefaultConnection");
```

問題在啟動時就看到,而不是等使用者打到第一個請求。
