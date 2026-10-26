# eCommerce 微服務學習筆記

> 每章一份 `summary.md`:**一句話重點 → 為什麼 → 步驟 → 關鍵概念 → 踩坑 → 作業 / 下一步**。先看這頁的速查,細節點進章節。

## 學習地圖

| 章 | 主題 | 一句話 |
|---|---|---|
| [1_12](<1_12 Creating Users Microservice/summary.md>) | 建立專案結構 | Solution + 三專案,用專案參考定依賴方向,各層 DI 打包成擴充方法 |
| [2_13](<2_13_Exception Handling Middleware/summary.md>) | 例外 Middleware | 管線最前面的「總保全」:記 log、回乾淨的 500 |
| [3_14](<3_14_Create User Model in Core/summary.md>) | Entity 與 DTO | Entity = 資料表形狀,DTO = API 收發形狀 |
| [4_15](<4_15_Create User Repository in Core and Infra/summary.md>) | Repository | Core 放介面、Infrastructure 放實作(依賴反轉) |
| [5_16](<5_16_Create User Service/summary.md>) | Service | 商業邏輯集中,資料存取交給 Repository |
| [6_17](<6_17_Create_Users_Controller/summary.md>) | AuthController | 收 HTTP、呼叫 Service、回對的狀態碼 |
| [7_18](<7_18 Running API in https and Postman/summary.md>) | HTTPS、Postman、除錯 | 用 https profile 跑、Postman 測、VS Code F5 中斷點 |
| [8_19_20](<8_19_20_AutoMapper/summary.md>) | AutoMapper | 物件對應規則集中在 Profile |
| [9](<9_PostgreSQL_EFCore/summary.md>) | PostgreSQL + EF Core | Docker 跑資料庫、Migration、BCrypt 雜湊密碼 |
| [10](<10_FluentValidation/summary.md>) | FluentValidation | 進 Service 前擋壞資料,回 400 |
| [11](<11_Angular_Client_CORS/summary.md>) | Angular + CORS | 瀏覽器跨來源要後端允許 |
| [12](<12_Products_DAL/summary.md>) | Products:DAL | MySQL、Repository、Migration |
| [13](<13_Products_BLL_API/summary.md>) | Products:BLL + API | DTO、Service、Validator、Minimal API、Swagger、CORS |
| [13b](<13b_Angular_Products_Integration/summary.md>) | Angular 接兩個服務 | 前後端契約、憑證、Angular 一次請求怎麼走 |
| [14](<14_Docker_Intro/summary.md>) | Docker 介紹 | 為什麼需要、VM vs Container、Volume |

觀念比較(獨立檔案):
- [Singleton vs Scoped vs Transient](<4_15_Create User Repository in Core and Infra/Difference between Singleton and Scoped.md>)
- [為什麼 API 同時參照 Core 與 Infrastructure](<3_14_Create User Model in Core/Why API references Core and Infrastructure.md>)
- [基底 Image 與 Linux 的 Kernel / Userland](<14_Docker_Intro/Base Image and Linux Userland.md>)

## 架構速查

> **依賴方向永遠指向 Core**:`API → Core ← Infrastructure`
> Products 作業的對應:`APILayer → BLL → DAL`(BLL 直接參考 DAL,是作業的設計)。

| 層 | 放什麼 |
|---|---|
| Core(BLL) | Entities、DTOs、Service / Repository 介面、Services、Mappers、Validators |
| Infrastructure(DAL) | Repository 實作、DbContext、Migration、`DependencyInjection` |
| API | Controller / Minimal API、Middleware、`Program.cs` |

> 各層的 DI 註冊放在**各自的 `DependencyInjection` 擴充方法**,`Program.cs` 只呼叫。

### 命名慣例
| 前綴 | 意思 | 例 |
|---|---|---|
| `Add` | 註冊服務(DI) | `AddInfrastructure()` |
| `Use` | 加 Middleware | `UseCors()` |
| `Map` | 對應路由 | `MapControllers()`、`MapProductAPIEndpoints()` |

### 服務生命週期
| 註冊 | 存活 | 用在 |
|---|---|---|
| `AddTransient` | 每次新建 | 無狀態 Service |
| `AddScoped` | 一個請求內共用 | `DbContext`、Repository |
| `AddSingleton` | 一輩子一個 | Cache、設定 |

## 指令速查

```bash
# EF Core Migration(在 solution 根目錄)
dotnet ef migrations add <名稱> --project <含DbContext的專案> --startup-project <API專案>
dotnet ef database update      --project <含DbContext的專案> --startup-project <API專案>
dotnet ef migrations list      --project ... --startup-project ...   # 看有沒有 Pending

# HTTPS 開發憑證、以 https profile 執行
dotnet dev-certs https --trust
dotnet run --project <API專案> --launch-profile https

# Docker 資料庫
docker ps
docker exec -it <容器> psql -U postgres -d <db> -c '\dt'            # PostgreSQL
docker exec -it <容器> mysql -uroot -p <db> -e "DESCRIBE Products;"  # MySQL
docker volume ls
```

## 常見錯誤速查

| 症狀 | 先檢查 |
|---|---|
| 瀏覽器 `blocked by CORS policy`,但 Postman 正常 | `WithOrigins` 的值是否與前端 origin **逐字相同**(不能有結尾 `/`);`UseCors()` 位置 |
| `ERR_CERT_AUTHORITY_INVALID` | `dotnet dev-certs https --trust` |
| `Unknown column` / `Table doesn't exist` | `migrations list` 有沒有 Pending,跑 `database update` |
| `dotnet ef` 報 `entry point exited without ever building an IHost` | `Program.cs` 有沒有 `Build()` / `Run()` |
| 改了程式碼沒生效 | 重啟 API(或 `dotnet watch run`) |
| 登入 401 / 回應 `status = 0` | 401 = 到達 API 但登入失敗;0 = 被 CORS 擋 |
| `Table doesn't exist`(MySQL 在 Docker) | Linux 表名區分大小寫,用 `Products` 不是 `products` |
| 容器重建後資料不見 | 沒掛同一個 volume(用有名字的 volume) |

## 待辦
- UserService:JWT(Token 目前寫死 `"token"`)、Gender `Others` 對不上、前端錯誤顯示
- Products:授權(目前任何人都能 POST / PUT / DELETE)
- Docker 化 UserService 與 Products(Dockerfile + docker-compose)
- Angular Step 2:表單
