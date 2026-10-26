# 13b Angular 接兩個微服務(UserService + ProductsService)

> **一句話**:前後端共同遵守的是「契約」(URL、HTTP 方法、JSON 欄位名);前端是老師給的已知契約時,改後端對齊,不改前端。

## 步驟
1. 下載老師**新版** Angular(多了 `products.service.ts`、商品元件、`isAdmin`),複製進專案時**保留 `.git`**
2. `src/environment.ts` 改成自己的 HTTPS port(結尾要 `/`):

   | 服務 | 網址 |
   |---|---|
   | UserService | `https://localhost:7032/api/auth/` |
   | ProductsService | `https://localhost:7046/api/products/` |

3. 比對前端實際打的 URL:搜尋是 `GET {base}search/{searchString}`,與 API 不同 → **改 API 路由**並同步 README
4. `dotnet dev-certs https --trust`
5. 灌入範例資料(表名 `Products`,MySQL 在 Linux **區分大小寫**)
6. 啟動:UserService(PostgreSQL 容器)、ProductsService(MySQL 容器)、`ng serve`

## 關鍵概念

> **自簽憑證與 `curl -k`**
> `-k` 是使用者自己選擇略過檢查。瀏覽器裡的 JS 無權略過,要用 `dotnet dev-certs https --trust` 讓系統信任。

> **Admin 寫死在前端**
> `admin@example.com` / `admin` 在 Angular 的 `users.service.ts`,不在資料庫,也沒走後端。隱藏按鈕只是「體驗」,不是「安全」:Products API 目前沒有授權,任何人都能 `POST` / `PUT` / `DELETE`。之後 JWT / Entra ID 要補。

> **這個 Angular 用什麼 UI**
> Angular 17 + **Angular Material**(indigo-pink 主題)。用到 `mat-card`、`mat-form-field`、`mat-select`、`mat-table`、`mat-icon`、`mat-toolbar` 等。

## 一次請求怎麼走完(Angular 學習 Step 1)

```
網址 /admin/products
  → app.routes.ts 對應到 ProductsComponent
  → constructor 注入 ProductsService
  → ngOnInit() 呼叫 getProducts().subscribe(...)
  → HttpClient 發 GET .../api/products/
  → 回應回來,next 裡 this.products = response
  → .html 的 <table [dataSource]="products"> 自動重畫
```

| 概念 | 重點 |
|---|---|
| Routing | 網址 → component(類似 `MapGet`) |
| 建構子注入 | 和 .NET 的 DI 同一想法;`providedIn: 'root'` 類似 `AddSingleton` |
| `ngOnInit` | component 建好後執行一次,適合載入資料 |
| `Observable` | **不 `subscribe` 就不會真的發請求**(.NET 的 `Task` 一呼叫就開始) |
| 非同步 | `subscribe` 後面那行會**立刻**執行,資料要在 `next` 裡才拿得到 |

## 踩過的坑
- `environment.ts` 還是老師的 port(7186 / 7260),要改成自己的
- 搜尋路徑與前端不同(作業文字 `/products/search/`,前端 `/search/`)

## 資料庫:Products 範例資料與長度
- 老師的 12 筆範例資料是給他的 `products` 小寫表;你的表是 `Products`
- 直接灌 SQL 會繞過 FluentValidation;資料庫約束是最後防線
- 驗證長度要**小於等於**資料庫欄位長度,否則漏到資料庫變成 500

## 還沒做(Angular)
- 動手題:加 `console.log`、暫時註解 `subscribe`,看 Network 有沒有請求
- 小問題:`products: ProductResponse[] = []` 為什麼要有初始值?
- Step 2:表單(`new-product` 如何收集輸入、驗證、顯示錯誤)
