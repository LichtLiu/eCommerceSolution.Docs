# Transient / Scoped / Singleton(服務生命週期)

> **一句話**:生命週期決定「注入的物件何時建立、何時銷毀」。

| | `AddTransient` | `AddScoped` | `AddSingleton` |
|---|---|---|---|
| 存活 | 最短,每次要用就新建 | 一個 HTTP 請求內共用 | 程式啟動到關閉,只有一個 |
| 比喻 | 免洗筷 | 旅客專屬房間 | 飯店大廳的大時鐘 |
| 適合 | 無狀態的商業邏輯(`UsersService`) | `DbContext`、`Repository` | Cache、設定檔、記憶體暫存資料 |

## 該怎麼選

> **DbContext 與 Repository 用 Scoped**
> `DbContext` 不是 thread-safe,而且會追蹤狀態。Singleton 會讓所有請求共用同一個,並行時出錯(甚至連線永遠不關);Scoped 每個請求一個,互不干擾。

> **Captive Dependency(被綁架的相依)**
> Singleton 注入了 Scoped 的東西,那個 Scoped 物件會被綁成跟 Singleton 一樣久。.NET 在開發模式會驗證並丟例外,所以「生命週期要一致」。

## 本專案的演進

| 階段 | 設定 | 原因 |
|---|---|---|
| 一開始 | Repository 用 Singleton | 還沒接資料庫,靠 Singleton 讓記憶體裡的 List 不會消失 |
| 接上 EF Core 後 | 改成 Scoped | `DbContext` 是 Scoped,Repository 要一致 |
