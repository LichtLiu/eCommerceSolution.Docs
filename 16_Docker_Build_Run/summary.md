# 16 `docker build` 與 `docker run`

> **一句話**:`docker build` 依 Dockerfile 建 **Image**,`docker run` 依 Image 建並跑 **Container**;要讓主機連得到,要加 `-p` 做 Port 映射。

## 1. 為什麼需要
Dockerfile 只是腳本。要真的產生 Image、跑出容器,靠這兩個指令。

## 2. 步驟

### ① 建立 Image
```bash
docker build -t mydotnetapp:initial -f ProductsMicroservice.API/Dockerfile .
```

| 部分 | 意思 |
|---|---|
| `-t 名稱:tag` | 幫 Image 取名;tag 通常是版本號或特殊名稱;一個 Image 可有多個 tag |
| `-f 路徑` | Dockerfile 位置。**就在目前資料夾時可省略**(會自動找);在子資料夾就要明寫(資料夾/檔名) |
| `.`(最後) | **Docker context**:目前的 solution 資料夾,`COPY` 能複製的範圍 |

- 第一次 build 會下載基底 Image(`sdk`、`aspnet`),老師說約 30~40 秒;本機已有就直接用,更快。
- `docker images`:列出本機所有 Image(老師一度說成 `docker ls`,自己更正)。
- 老師的 Image 約 **238 MB**:只含 Runtime 與相依,**沒有原始碼**。

### ② 建立並執行 Container
```bash
docker run -p 8080:8080 -p 8081:8081 mydotnetapp:initial
```

| 部分 | 意思 |
|---|---|
| `-p 主機Port:容器Port` | Port 映射(可寫多次);左外右裡 |
| `名稱:tag` | **兩個都要寫,沒有 tag 不行**;也可改用 Image ID(十六進位) |

> 沒有 Port 映射,應用程式在容器裡跑著,但**主機上的 client 連不進去**。

### ③ 老師的結果
| 項目 | 結果 |
|---|---|
| 容器啟動 | 成功,應用聽 8080 |
| 瀏覽器 `localhost:8080` | 無錯誤 |
| 打 `api/products` | 失敗:`unable to connect to any of the specified mysql hosts`,容器裡沒有 MySQL 服務 |
| 後續 | 之後的課會教怎麼把資料庫服務加進來 |

- GUI(Docker Desktop)可看 Image、Container、Port,也能停止與刪除。
- 實務上用 Visual Studio 選 **Docker profile** 就能直接執行,內部用的就是這些指令。

## 3. 關鍵概念

| 概念 | 說明 |
|---|---|
| Docker context | `docker build` 最後的 `.`;`COPY` 只能複製 context 裡的東西 |
| `-f` 與 context 是兩件事 | `-f` = Dockerfile 在哪;`.` = 能複製哪些檔案 |
| tag 必須寫 | 沒寫 tag 的 `docker run` 對老師的 Image 不行 |
| Port 映射 | 主機 Port 只能綁一次(見 15_Dockerfile 的 Port 問答) |
| 錯誤訊息 | `unable to connect to ... mysql hosts` = 容器跑著、但連不到資料庫 |

> 補充(我的知識,未驗證):容器裡的 `localhost` 指的是**容器自己**,不是我的 Mac,也不是另一個容器。所以我的 API 容器的連線字串不能寫 `Server=localhost`;要連另一個容器的資料庫,要靠 Docker 網路(docker-compose 會處理)。連線字串用環境變數注入(見 14 的 Do not bake secrets)。

## 4. 作業(自己練習)
1. 為什麼 `-f` 要寫路徑,而最後的 `.` 是 solution 資料夾?兩者各是什麼?
2. 若 Dockerfile 就在 solution 根目錄,`docker build` 要怎麼寫?
3. 為 Products 服務寫第一版 Dockerfile,`docker build -t products-api:initial ...`,看 `docker images` 的大小,再 `docker run -p ...` 試試。
4. 預期會遇到什麼錯誤?(提示:連線字串、容器的 `localhost`)

## 實作:我的 Products 服務(`eCommerceSolution.ProductsService`)

- 檔案:solution 根目錄的 `Dockerfile` 與 `.dockerignore`(context = 根目錄)。
- 四階段:`base`(aspnet:10.0)→ `build`(sdk:10.0)→ `publish` → `final`。先 COPY 三個 csproj 再 `restore`,再 COPY 全部原始碼。
- `.dockerignore` 排除 `bin/`、`obj/`、`.git` 等:否則 `COPY . .` 會把主機的 `bin/obj` 帶進去,可能干擾建置。
- 容器內 Kestrel 聽 `8080`(`ENV ASPNETCORE_URLS=http://+:8080`),沒有 HTTPS(沒有憑證)。

```bash
cd eCommerceSolution.ProductsService
docker build -t products-api:initial .
docker run -d --name products-api -p 8080:8080 \
  -e "ConnectionStrings__DefaultConnection=Server=host.docker.internal;Port=3306;Database=ecommerce_products;User=root;Password=<密碼>" \
  products-api:initial
```

**實測結果**
| 項目 | 結果 |
|---|---|
| build | 成功,約 56 秒(含第一次下載基底 Image) |
| Image 大小 | **248 MB** |
| 成品裡有 SDK 嗎 | **沒有**(`dotnet --list-sdks` 無輸出;只有 Runtime 10.0.14) |
| 執行身分 | `uid=1654(app)`,不是 root |
| Swagger / API | `localhost:8080/swagger` 200;`GET /api/products` 200,取回 14 筆;`search/Samsung` 2 筆 |

**踩到的坑**
- 容器的連線字串不能用 `localhost`(那是容器自己)。用環境變數 `ConnectionStrings__DefaultConnection` 覆蓋,`Server=host.docker.internal` 能連到 Mac 上對外映射的 MySQL(我的環境實測可行)。
- 驗證到一半 API 一直 `Connection refused`,原因是 **`products-mysql` 與 `ecommerce-postgres` 兩個容器都停了**(`Exited (255)`,log 最後是 `Received SHUTDOWN from user via user signal`,像是 Docker Desktop 重啟),不是 Dockerfile 的問題。`docker start` 後資料都在。排查順序:先 `docker ps -a` 確認 DB 容器還在跑。
- 兩個資料庫容器沒設重啟策略,Docker 重啟後不會自動起來。可加 `--restart unless-stopped`(我的知識,尚未套用)。
- log 有 AutoMapper 授權金鑰的警告(商業授權,學習用可忽略)。

### 問:Dockerfile 搬到 `APILayer/`,仍在 solution 根目錄 build,指令怎麼寫?`COPY` 路徑要改嗎?

> **我的答案**:`docker build -t image-is-me:tag-is-me -f ./APILayer/Dockerfile .`;`COPY` 路徑要不要改,不知道。

**補充**:指令正確,**`COPY` 路徑不用改**。

> `COPY` 的來源路徑是相對於 **context(最後的 `.`)**,不是相對於 Dockerfile 所在的位置。
> | 參數 | 回答的問題 |
> |---|---|
> | `-f ./APILayer/Dockerfile` | **腳本在哪裡?** |
> | 最後的 `.` | **`COPY` 可以從哪裡複製?**(context) |

**實測**(暫存副本):

| 情境 | 結果 |
|---|---|
| Dockerfile 搬到 `APILayer/`,context = `.`,`COPY` 不改 | **成功**,Image ID 與放根目錄時**完全相同**(Dockerfile 位置不影響 Image 內容) |
| 同樣的 Dockerfile,context 改成 `APILayer` | **失敗**:`"/APILayer/APILayer.csproj" not found`、`"/DataAccessLayer/DataAccessLayer.csproj" not found`、`"/BusinessLogicLayer/..." not found` |

- 老師的 Dockerfile 放在 API 專案資料夾,是 Visual Studio「Add Docker Support」的習慣(放在要容器化的專案裡),不是規定。他截圖第 13~16 行路徑開頭都是 `ProductsMicroService.API/`,證明 context 是 solution 根目錄。
- 只對 API 的 csproj 下 `build` / `publish` 也會連 BLL、DAL 一起帶進去(`ProjectReference` 自動編譯與發佈):我的 Image 的 `/app` 裡有 `APILayer.dll`、`BusinessLogicLayer.dll`、`DataAccessLayer.dll`。
- `.dockerignore` 要放在 context 根目錄(我的知識,未專門驗證)。
- 名稱對照:老師的 `ProductsMicroService.API` = 我的 `APILayer`。

### 實驗:`docker build` 的快取(改 `Program.cs` 再 build)

build 輸出裡 `CACHED` 代表沿用快取。實驗結果(只改 `Program.cs`):

| 步驟 | 狀態 |
|---|---|
| `restore` | **CACHED**(csproj 沒變) |
| `COPY . .` | **重新執行**(原始碼內容變了) |
| `build`(7.1s)、`publish`(6.8s) | 重新執行,排在 `COPY . .` 後面 |
| `base`、`final` 的 `WORKDIR` | CACHED(另一個階段,不受影響) |

> **某一步的輸入變了,這一步就重做,它後面的步驟也全部重做。** 所以 Dockerfile 要把「很少變的」(csproj → restore)放前面、「常變的」(全部原始碼)放後面。
> 其他觀察:`.dockerignore` 讓 context 只有 30.89kB;`products-api:initial` 與 `:v2` IMAGE ID 相同,**tag 只是同一個 Image 的多個名字**。

### 問:連線字串為什麼要寫 `host.docker.internal`,不能寫 `localhost`?我怎麼知道的?

> **我的答案**:不知道。

**補充**:每個容器有自己的網路空間,**容器裡的 `localhost` 是容器自己**。`host.docker.internal` 是 **Docker Desktop(Mac / Windows)內建的特殊名稱**,指到主機(Linux 直接裝 Docker 時預設沒有,我的知識)。我的環境實測(在容器裡解析):

| 在容器裡查 | 結果 | 意思 |
|---|---|---|
| `localhost` | `::1` | 容器自己 |
| `host.docker.internal` | `192.168.65.2` | Docker Desktop 設定的主機位址 |
| API 容器自己的 IP | `172.17.0.4` | |
| `products-mysql` 的 IP | `172.17.0.2` | |

第一次測試連線被拒絕,是因為當時 `products-mysql` 與 `ecommerce-postgres` 兩個容器停了(像是 Docker Desktop 重啟),不是名稱的問題。

### 問:為什麼不直接寫 `Server=172.17.0.2`?容器刪掉重建,IP 會重設嗎?

> **我的答案**:會。

**實驗**(兩個測試容器,只改啟動順序,已清掉):

| 啟動順序 | `ipdemo1` | `ipdemo2` |
|---|---|---|
| 第一次:1 → 2 | `172.17.0.5` | `172.17.0.6` |
| 全部停掉再反過來:2 → 1 | **`172.17.0.6`** | **`172.17.0.5`** |

> **容器的 IP 是啟動時由 Docker 從網段依序發下去的,不是容器的屬性。** 停掉再啟動就可能換,連線字串寫死 IP 會悄悄失效。
> 我的 `products-mysql` 現在還是 `.2` 只是巧合(我重啟時它是第一個起的),不是保證。
> 正確做法:用**名稱**(`host.docker.internal`,或之後 Docker 網路 / compose 的容器名稱,我的知識,課程未確認),不要用 IP。

### 問:換成用容器名稱連線,要改的是 `appsettings.json`、Dockerfile,還是只改 `docker run` 的 `-e`?

> **我的答案**:只改 `docker run` 的 `-e`,這樣就不用為了改連線重新打包。

| | 連線字串寫進 appsettings / Dockerfile | 執行時用 `-e` 給 |
|---|---|---|
| 換環境 | 改檔案、**重新 build 新 Image** | 只改指令,Image 不動 |
| Image 數量 | 每個環境一個 | **一個 Image 跑所有環境** |
| 祕密 | 被打包進 layer | 不在 Image 裡 |
| 可追查性 | 設定一改就是沒測過的新 Image | 測過的 Image 就是上線的 Image,環境間只有設定不同 |

> **Build once, run anywhere**:Image 是不變的成品,設定在執行時才注入。

### 問:容器 `Exited (0)` 時,可寫層的資料還在嗎?什麼時候才會消失?

> **我的答案**:資料還在,`docker rm` 才把整個實例砍掉。

```
docker run    → 建立容器 + 啟動
docker stop   → 停止行程,容器還在(可寫層還在) → docker start 可繼續用
docker rm     → 刪掉容器,可寫層跟著消失
```

| 資料放哪 | `stop` | `rm` |
|---|---|---|
| 容器可寫層 | 還在 | **消失** |
| Named volume | 還在 | **還在**,掛回去即可 |
| 匿名 volume(`products-mysql`) | 還在 | 通常還在,但名字是亂碼難接回;`docker rm -v` 會連它刪掉(我的知識,未執行) |

> `stop` 是關機,`rm` 是把整台機器砍掉。`rmi` 刪的是 Image,不是容器。
> `Exited (N)`:N 是主程式的結束代碼。0 = 正常結束(`docker stop` 後 ASP.NET 優雅關閉);非 0 = 出錯或被強制停止(`255` 的原因我不確定)。排查順序:`docker ps -a` 看狀態 → `docker logs`。
> 狀態在哪看:`docker ps`(只看執行中)、`docker ps -a`(含已停止)、`docker inspect`(詳細)、Docker Desktop。
> 刪除順序:先 `docker rm`(容器),再 `docker rmi`(Image);有容器在用 Image 就刪不掉,不要用 `-f`。

## 5. 結果與下一步
- 已上完:`docker build`、`docker run`、Port 映射、為什麼連不到資料庫。
- 下一步:把資料庫服務加進容器 / 容器之間互連。
