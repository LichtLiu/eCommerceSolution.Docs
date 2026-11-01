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

另外:使用者名稱是**公開**的,會出現在 Image 全名裡;帳號密碼自己保管。

## 4. 作業(自己練習)
1. 註冊 Docker Hub、建一個 repository,名稱與 tag 規則自己決定。
2. `docker login`,看到 `Login succeeded`。
3. 為什麼推 Image 之前要先處理 `appsettings.json` 裡的密碼?
4. (下一堂)為本機 Image 加上 `帳號/repository:tag` 的名稱,再 `docker push`。

## 5. 結果與下一步
- 已上完:Docker Hub 帳號、repository、tag、登入。
- 下一堂:建 Image 並 push 到 repository。
