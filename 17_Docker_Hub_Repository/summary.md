# 17 Docker Hub 帳號與 Repository

> **一句話**:Repository 是「一個專案的 Image 集合」,每個版本用一個 **tag** 表示;要把 Image 分享出去,先有帳號與 repository,並且本機要先 `docker login`。

## 1. 為什麼需要
前面學了 `docker push` / `docker pull`。要把自己的 Image 推上去,要有帳號、有 repository、本機登入。

## 2. 步驟(老師示範)

| # | 做什麼 |
|---|---|
| 1 | 到 hub.docker.com 註冊(Google 或 Email),取**不重複的使用者名稱** |
| 2 | 選方案:學習與練習選 **Personal(免費)** |
| 3 | 收驗證信,點連結驗證 Email |
| 4 | Repositories → Create:名稱**全小寫**,可含連字號,不可有空格;選 public / private |
| 5 | 建好後頁面會顯示 `docker push` 的指令範例 |
| 6 | 本機登入:Docker Desktop 右上角 **Sign in**,或終端機 `docker login` |
| 7 | 登入成功顯示 `Login succeeded`;**沒登入就不能 push** |

Docker Desktop 的 **Local** 分頁看本機 Image,**Hub** 分頁看遠端的。

## 3. 關鍵概念

> **Repository = 一組 Image 的集合,每個 Image 用一個 tag 表示。**
> ```
> mysql(repository)
>  ├─ 8.3.0(tag = 一個 Image)
>  ├─ 8.0(tag = 另一個 Image)
>  └─ ...
> ```

| 名詞 | 說明 |
|---|---|
| 完整名稱 | `帳號(namespace)/repository 名稱`,例如 `<帳號>/ecommerce-products-microservice` |
| 慣例 | 一個專案一個 repository,**一個版本一個 tag**(1.0、2.0) |
| `docker login` | 一次只能登入一個帳號(老師說) |

**老師講的規則(以 Docker Hub 官方為準,我無法驗證目前規定)**:
- 使用者名稱:4 到 13 個字元、只能字母與數字、不能有連字號
- 免費帳號:1 個 private repository、public 不限數量
- public:所有人可看、可下載;private:只有自己
- 自動 build(推 GitHub 自動打包):老師說只有付費版

## 注意:Image 裡的祕密(我實際查過)

我的 `products-api:initial` 裡 `/app/appsettings.json` 有 `Password=<測試密碼>`(`docker run --rm --entrypoint cat products-api:initial /app/appsettings.json` 看得到)。

> **推到 public repository 之前,一定要先把祕密移出 Image**:連線字串不含密碼,密碼靠 `-e` / `--env-file` 在執行時給。見 [14 的 Do not bake secrets into the Image.md](<../14_Docker_Intro/Do not bake secrets into the Image.md>)。

### 處理方式(已完成,Products 專案)

| # | 改什麼 | 為什麼 |
|---|---|---|
| ① | `dotnet user-secrets init` + `set "ConnectionStrings:DefaultConnection" "..."`(`--project APILayer`) | 本機開發要有地方放密碼。user-secrets 存在 Mac 的使用者資料夾,不進 git、不會被 `COPY` 進 Image;`APILayer.csproj` 多一行 `UserSecretsId` |
| ② | `appsettings.json` **移除 `ConnectionStrings` 整段** | Image 裡就沒有密碼 |
| ③ | `DataAccessLayer/DependencyInjection.cs`:讀不到就丟 `InvalidOperationException`(fail fast) | 沒有連線字串時啟動就明確失敗,不要等第一個請求才 500 |
| ④ | README:改成「怎麼提供連線字串」,移除 README 裡的密碼,加「Run with Docker」 | 別人 clone 後知道怎麼跑 |

**驗證結果**

| 驗證 | 結果 |
|---|---|
| 本機 `dotnet run`(Development,讀 user-secrets) | 正常,取回 14 筆 |
| `dotnet ef migrations list` | 讀得到連線,列出 `InitialCreate` |
| 新 Image `products-api:v3` 的 `/app/appsettings.json` | 只剩 Logging 與 AllowedHosts;整個 `/app` 搜尋 `mysql123` **沒有** |
| 不給連線字串就 `docker run` | **啟動就失敗**,訊息 `Missing connection string 'ConnectionStrings:DefaultConnection'...`,容器 `Exited (139)`(非 0) |
| `docker run --env-file <檔案>` | 200,14 筆;`search/Samsung` 2 筆;`docker inspect` 的指令與參數裡**沒有**密碼 |

> **做法分工**:本機 → user-secrets;容器 → 環境變數 `ConnectionStrings__DefaultConnection`(`-e` 或 `--env-file`,檔案放在 repo 外面);Image 裡只放不含祕密的設定。
> 限制:這個密碼**仍在 Git 歷史裡**(舊 commit 的 `appsettings.json`),而且 `docker inspect` 看得到容器的環境變數(我的知識,未實測)。這是本機測試密碼所以風險小;真正的祕密要輪替,正式做法是平台的 secret 機制(後面課程)。
> `--env-file` 的格式是 `名稱=值`,不要加引號。

另外:使用者名稱是**公開**的,會出現在 Image 全名裡;帳號密碼自己保管。

## 4. 作業(自己練習)
1. 註冊 Docker Hub、建一個 repository,名稱與 tag 規則自己決定。
2. `docker login`,看到 `Login succeeded`。
3. 為什麼推 Image 之前要先處理 `appsettings.json` 裡的密碼?
4. (下一堂)為本機 Image 加上 `帳號/repository:tag` 的名稱,再 `docker push`。

## 5. 結果與下一步
- 已上完:Docker Hub 帳號、repository、tag、登入。
- 下一堂:建 Image 並 push 到 repository。
