# 15 Dockerfile:多階段建置(multi-stage)

> **一句話**:Dockerfile 是建立 Image 的腳本,每道指令疊一個 layer;**多階段**讓你先用 SDK 編譯、發佈,最後只把**發佈結果**放進 Runtime 的 Image,所以成品小、乾淨。

## 1. 為什麼需要
- 要有一份可重複的「建 Image 步驟」,而不是手動裝環境。
- 編譯需要 SDK、NuGet 套件、原始碼,**執行用不到**。全部留在最終 Image 會又大又不安全。

| 重點 | 說明 |
|---|---|
| 檔名固定 | 就叫 `Dockerfile`,Docker Engine 用預設名稱自動找到它,**不要改名**(想用別的檔名要另外指定,我的知識) |
| 誰執行 | `docker build` → Docker Engine(daemon)依序執行 |
| 每道指令 | 疊一個 layer,所有 layer 合起來 = Image |
| 怎麼來的 | 老師在 Visual Studio 建立專案時勾選 **Use Docker**,會自動產生 |

## 2. 步驟:一份 Dockerfile 怎麼讀

> 下面是依老師講解**還原**的版本(以他的 Products API、.NET 8 為例),**不是逐字內容**;套件名稱與路徑以自己的專案為準。

```dockerfile
# ── 階段 1:base(只是執行環境,先放著)──────────────
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
USER app
WORKDIR /app
EXPOSE 8080
EXPOSE 8081

# ── 階段 2:build(用 SDK 編譯)─────────────────────
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
ARG BUILD_CONFIGURATION=Release
WORKDIR /src
COPY ["ProductsMicroService.API/ProductsMicroService.API.csproj", "ProductsMicroService.API/"]
RUN dotnet restore "ProductsMicroService.API/ProductsMicroService.API.csproj"
COPY . .
WORKDIR "/src/ProductsMicroService.API"
RUN dotnet build "ProductsMicroService.API.csproj" -c $BUILD_CONFIGURATION -o /app/build

# ── 階段 3:publish(發佈,產生可執行的檔案)──────────
FROM build AS publish
ARG BUILD_CONFIGURATION=Release
RUN dotnet publish "ProductsMicroService.API.csproj" -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false

# ── 階段 4:final(最終 Image)───────────────────────
FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "ProductsMicroService.API.dll"]
```

### 四個階段在做什麼

| 階段 | 基底 | 做什麼 | 進最終 Image 嗎 |
|---|---|---|---|
| `base` | `aspnet:8.0`(Runtime) | 設使用者、工作目錄、開放 Port | **是**(`final` 以它為基底) |
| `build` | `sdk:8.0`(SDK) | 複製 csproj → restore → 複製全部原始碼 → build | 否 |
| `publish` | `build` | `dotnet publish` 產生最佳化的發佈檔 | 否,只有發佈檔被複製走 |
| `final` | `base` | 把 `/app/publish` 複製進來,設啟動指令 | **這就是成品** |

> **為什麼要多階段**:暫存檔(NuGet 套件、編譯過程檔、原始碼)不留在最終 Image,只留執行所需的發佈檔,**Image 大小少很多**。

> **怎麼理解「只把發佈結果放進 Runtime 的 Image」:做菜與上菜**
> | 階段 | 比喻 | 對應 |
> |---|---|---|
> | 做菜(build / publish) | 廚房:砧板、刀、食材、瓦斯爐 | `sdk` Image:SDK、原始碼、NuGet 套件 |
> | 成品 | 煮好的那道菜 | 發佈結果(`/app/publish`,編譯好的 dll) |
> | 上菜(final) | 餐桌 + 盤子,只放菜 | `aspnet` Image(Runtime)+ 發佈結果 |
>
> 客人只要吃到菜,不需要把整間廚房搬到餐桌上。`COPY --from=publish` 就是「把菜從廚房端到餐桌」,廚房階段用完就丟。
>
> .NET 的角度:SDK 像剪輯軟體、原始碼與 NuGet 套件像剪輯專案檔與素材、發佈結果(dll)像匯出的影片、Runtime 像播放器。**最終 Image = 播放器 + 影片**。
>
> 一句話:多階段 = 在一個 Image 裡做完菜,但出貨時只端菜、不端廚房。

### 問:不用多階段、全程用 `sdk` 也能跑,為什麼還要多階段?

> **我的答案**:因為這樣就等於把整個廚房用具搬出來,像 NuGet package、SDK 等等。

| | 全程 `sdk` | 多階段 |
|---|---|---|
| 最終 Image 裡有 | SDK、NuGet 套件、**原始碼**、**編譯過程檔**、發佈結果 | 只有 Runtime + 發佈結果 |
| 體積 | 大 | 小 |
| 安全 | 攻擊面大,有編譯器和原始碼 | 攻擊面小 |
| 能不能跑 | 能 | 能 |

> 補充:容易被忽略的是**原始碼**和**編譯過程檔**也會跟著出貨。
> 面試版:「能跑,但最終 Image 會帶著 SDK、NuGet 套件和原始碼,又大又不安全。多階段讓我在 build 階段用 SDK 編譯,最後只把發佈結果放進只有 Runtime 的 Image。」

### 指令表

| 指令 | 作用 |
|---|---|
| `FROM <image>:<tag> AS <別名>` | 指定基底 Image(來自 registry,如 Docker Hub 或 `mcr.microsoft.com`);別名供後面引用 |
| `USER app` | 用自訂使用者 `app` 執行,**不要用 root**(安全) |
| `WORKDIR /app` | 設定工作目錄,之後指令都在這個路徑下執行(像 `cd`);和使用者 `app` 同名只是巧合,無關 |
| `EXPOSE 8080` / `8081` | **標示**要用的 Port(HTTP / HTTPS),**不會真的啟動伺服器**;只是宣告 |
| `ARG BUILD_CONFIGURATION=Release` | 建置時的變數與預設值,`docker build` 時可覆寫;**每個階段要各自宣告** |
| `COPY <來源> <目的>` | 來源是**主機**(你的電腦),目的是 Image 內的 Linux 檔案系統 |
| `RUN <指令>` | 建置時執行指令,像在終端機輸入 |
| `ENTRYPOINT ["dotnet", "xxx.dll"]` | 容器啟動時執行的指令,JSON 陣列:第一個是執行檔,後面是參數 |
| `COPY --from=<階段>` | 從別的階段複製檔案 |

## 3. 關鍵概念

> **先 COPY csproj、跑 `dotnet restore`,再 COPY 全部原始碼**
> `restore` 只需要 csproj 就能下載 NuGet 套件。老師的說明是:csproj 是安裝套件與編譯必需的。
> 補充(我的知識,老師沒提):這樣排序,原始碼改了但套件沒變時,`restore` 那層可以沿用快取,不必重新下載。

| 概念 | 說明 |
|---|---|
| `dotnet build` vs `dotnet publish` | `build` 編譯成 dll;`publish` 在 `build` 之後,產生**最佳化、可直接執行**的檔案 |
| `-c Release` vs `Debug` | Release 沒有中斷點等除錯符號,用 docker 執行不需要;要除錯才用 Debug |
| `/p:UseAppHost=false` | 發佈成**依賴框架(framework-dependent)**,不產生自帶的可執行檔;容器內用 `dotnet xxx.dll` 執行 |
| `sdk` vs `aspnet` | SDK 能編譯;`aspnet` 只含 Runtime。多階段就是 build 用前者、執行用後者 |
| tag | 同名 Image 可有不同 tag(例如 `8.0`、`9.0`),可到 MCR 網站查 |
| 預先安裝的好處 | 基底 Image 已含 Linux 與 ASP.NET Runtime,**不用手動安裝** |

> **對照之前的問答**:「最後的 Image 裡面還有 SDK 嗎?」沒有。`final` 是從 `base`(Runtime)開始,只複製 `/app/publish`,`build` 與 `publish` 階段的 layer 不在成品裡。老師這堂證實了。

### 問:`EXPOSE` 不會啟動伺服器,那 Port 能不能被主機連到,取決於什麼?

> **我的答案**:取決於 host 有沒有 listening?

**補充與修正**:要分兩層看。Host 端的 listening 是由 **`-p` Port 映射**建立的,不是 `EXPOSE`。

```
Mac(Host) ──→ [-p 映射] ──→ 容器裡的 Port ──→ 程式有沒有在 listen
     ①            ②                                ③
```

| 條件 | 說明 |
|---|---|
| ③ 容器裡的程式有在 listen | `EXPOSE` 不會讓程式去 listen |
| ② `docker run -p 主機Port:容器Port` | 沒有映射,主機就連不到(Port mapping,老師之後會講) |

**我的 `products-mysql` 就是證據**(`docker inspect` / `docker ps`):

| 項目 | 結果 | 意思 |
|---|---|---|
| exposed | `3306`、`33060` | MySQL Image 用 `EXPOSE` 標示的 |
| `0.0.0.0:3306->3306/tcp` | 有映射 | 我寫了 `-p 3306:3306`,主機連得到 |
| `33060/tcp` | **沒有箭頭** | 有 `EXPOSE`、沒映射,主機連不到 |

> **一句話**:`EXPOSE` 只是標示。要讓主機連到容器,要用 `-p` 映射,且容器裡的程式要真的在那個 Port listen。
> 補充(我的知識,老師未講):`EXPOSE` 是給人看的文件,也有工具(如 `docker run -P`)會依它自動映射;同一個 Docker 網路內的容器彼此連容器 Port,不需要 `-p`。

### 問:`-p 5433:5432` 時,Mac 要用哪個 Port?容器裡 PostgreSQL 聽哪個?

> **我的答案**:Mac 用 5432、容器裡用 5433。(**答反了**)

> **正確**:`-p 主機Port:容器Port`,**左邊 = 外面(主機),右邊 = 裡面(容器)**。
> `-p 5433:5432`:Mac 用 **5433**,容器裡的 PostgreSQL 仍聽 **5432**。

```
Mac:5433  ──→  [-p 5433:5432]  ──→  容器:5432(PostgreSQL 在這裡 listen)
```

| 用途 | 做法 |
|---|---|
| Mac 上已有 PostgreSQL 佔 5432 | 容器仍聽 5432,映射到主機 5433,避免衝突 |
| 同時開兩個 PostgreSQL 容器 | 兩個容器內都聽 5432,主機分別用 5433、5434 |

> 容器裡的設定不用改,只改外面的映射。`docker ps` 的 `0.0.0.0:5432->5432/tcp` 也是同一個方向:主機 → 容器。
> 若真的改映射,UserService 連線字串的 `Port=` 要跟著改成主機那個。改映射通常要重建容器(我的知識);資料在有名字的 volume,重建後掛回即可,**不要隨手 `docker rm`**。

### 問:兩個 MySQL 容器同時跑,為什麼不衝突?兩個都寫 `-p 3306:3306` 會怎樣?

> **我的答案**:`mysql-test` 的 host 是 3307、`products-mysql` 是 3306,不衝突。兩個都一樣會發生什麼,我不知道。

**實測**(用 `--rm` 的測試容器,失敗後無殘留,`products-mysql` 沒受影響):

```
docker: Error response from daemon: driver failed programming external connectivity
on endpoint port-conflict-test (...): Bind for 0.0.0.0:3306 failed: port is already allocated.
```

| 設定 | 結果 |
|---|---|
| `-p 3306:3306` + `-p 3307:3306` | 主機 Port 不同,**都能啟動**(容器內都聽 3306) |
| `-p 3306:3306` + `-p 3306:3306` | 主機 Port 相同,**第二個啟動失敗** |

> **衝突發生在主機那一側**:主機的 Port 是全機共用、只能綁一次;容器內的 Port 彼此隔離,不會衝突。

- 排查 `port is already allocated`:`docker ps` 看 `PORTS` 欄位;不是 Docker 佔的就可能是 Mac 本機服務(`lsof -i :3306`,我的知識,未執行);解法是換主機 Port。
- `docker run` 的完整寫法:`docker run -p 1234:33060 mysql:8`(選項在 Image 名稱前面);它是**建立新容器**,不是改現有容器的映射。
- 33060 是 MySQL X Protocol 的 Port(我的知識,未驗證),EF Core 連 MySQL 走 3306,不需要映射。

## 連回我的專案(我的知識,未驗證)
- 我的 Products 是**三個專案**(APILayer、BusinessLogicLayer、DataAccessLayer)。`dotnet restore` 需要被參考專案的 csproj 也在,所以要把**三個 csproj 都先 COPY**,Dockerfile 放在 solution 根目錄、build context 也從那裡。
- Port:我的 Products 在本機是 `5161` / `7046`(launchSettings)。在容器裡用哪個 Port、怎麼設定,等老師講 `docker run` 的 Port mapping。
- 連線字串別寫進 Image,見 [Do not bake secrets into the Image.md](<../14_Docker_Intro/Do not bake secrets into the Image.md>)。

## 4. 作業(自己練習)
1. 不看筆記,說明四個階段各自的基底 Image 與作用。
2. 把 `EXPOSE` 拿掉,Image 還能執行嗎?`EXPOSE` 到底有什麼用?(提示:老師說它不啟動任何伺服器)
3. 為什麼 `ARG BUILD_CONFIGURATION` 要在 `build` 與 `publish` 各宣告一次?
4. 為我的 Products 服務寫出第一版 Dockerfile(三個 csproj)。

## 5. 結果與下一步
- 已上完:Dockerfile 的指令與多階段建置。
- 下一堂:用 Docker 實際執行應用程式(`docker build`、`docker run`)。
