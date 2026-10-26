# 14 Docker 介紹:為什麼需要 Docker

> **一句話**:csproj 鎖住「程式用到的套件」,Docker 鎖住「程式跑的整個環境」。把程式連同它的執行環境打包成 Image,到哪都跑得一樣。

## 1. 為什麼需要:「在我電腦上可以跑」

程式在 **Dev → QA → Ops** 之間交付。原始碼不能單獨執行,還需要資料庫、Web 伺服器(Kestrel)、.NET 執行環境、NuGet 套件、環境變數。開發機都裝好了,測試與維運機器只有作業系統。

傳統做法是「編譯後的程式 + 安裝說明文件」,請對方照著裝:

| 問題 | 後果 |
|---|---|
| 手動安裝麻煩、易出錯 | 漏檔、少裝東西、人為失誤 |
| 版本不一致 | 開發用套件 6.0、測試時最新 6.1,內部變更導致啟動或執行期錯誤 |
| 責任歸屬爭吵 | 開發「我這邊可以跑」/ 測試「我照文件做了還是不行」 |

| 團隊 | 職責 |
|---|---|
| Dev | 寫程式、實作需求 |
| QA | 測功能與品質 |
| Ops | 維護基礎設施,確保正式環境穩定、可用、效能 |

## 2. 先前的解法:Hypervisor(虛擬機器)

> **Hypervisor**:讓同一台實體電腦建立多台虛擬機器(VM)的軟體,每台 VM 跑自己的作業系統。常見:VirtualBox、VMware、Hyper-V。

```
實體硬體
  └ Host OS
      └ Hypervisor
          ├ VM 1:Guest OS + 應用程式 + 相依套件
          ├ VM 2:Guest OS + 應用程式 + 相依套件
          └ VM 3:Guest OS + 應用程式 + 相依套件
```

把整個 VM 硬碟檔交給別的團隊就能跑,達成應用程式隔離。但有 4 個問題:

| 問題 | 說明 |
|---|---|
| 複製環境靠人工 | 每台 VM 都要裝 OS、伺服器、環境變數、函式庫、再放程式 |
| 體積與資源浪費 | 每台帶完整 Guest OS;資源預先配給,閒置也佔用 |
| 容易出錯 | 環境變數、相依軟體、設定、套件稍有不一致就出 bug |
| 難以 scale out | 10 個實例要 10 台 VM,要先有足夠硬體 |

> 例:16GB RAM 的 Mac 開 3 台各 4GB 的 VM,閒置也佔 12GB,最多只剩 4GB 給 macOS 和其他程式。

## 3. Docker 怎麼解決

| 名詞 | 是什麼 |
|---|---|
| **Image** | 範本:編譯後的程式 + 所有相依(伺服器、函式庫、.NET 執行環境、環境變數、指定版本套件),**不含完整作業系統** |
| **Container** | Image 的實例;可讀 Image 的檔案,也能存執行期變動(log、上傳檔);一個 Image 可建多個 |
| **Docker Engine(daemon)** | 執行 Container 的引擎 |

```
Docker Image(範本)
   ├→ Container 1
   ├→ Container 2
   └→ Container 3      ← 同一個 Image 的多個實例,彼此隔離
```

### VM vs Container
| | VM | Container |
|---|---|---|
| 作業系統 | 每台有自己的 Guest OS | **沒有**自己的 OS,用主機 OS 的 kernel |
| 資源 | 預先配給 CPU / RAM / 硬碟 | 不預先配給,直接用主機資源 |
| 主機角度 | 很重的 process(整台電腦) | 普通 process |
| 跑在 | Hypervisor 軟體上 | 容器化引擎(Docker daemon)上 |
| 體積 | 大(GB 級) | 輕量(約 100~200MB 等級,視內容) |
| 啟動 / 執行速度 | 慢 | **更快** |
| 隔離 | 有 | 有(差別在啟動時間、體積、硬體資源用量) |
| 擴展 | 貴 | 同一台主機輕鬆開多個實例,影響小 |

> **什麼時候選哪個**
> - 同一台硬體要跑**多種不同的作業系統** → VM
> - 應用程式隔離、部署簡單、啟動快 → Container
> - 兩者可以**合用**:在 VM 裡跑一個或多個 Docker 容器,多台 VM 跑在同一台主機上,兩邊的好處都拿到

> 補充(我的知識,未驗證):Mac 上的 Docker Desktop 是先開一台輕量 Linux VM,容器共用那台 Linux 的 kernel。所以 MySQL 容器內是 Linux,表名區分大小寫(`Products` vs `products`)。

### Image 與 Container 內部:分層檔案系統

```
Dockerfile(腳本)  →  Docker Engine 執行  →  Image(最終結果)
```

| 指令 | 作用 |
|---|---|
| `FROM` | 指定**基底 Image**(含特定 OS 版本的必要元件,有些預裝 .NET 等 SDK / Runtime) |
| `COPY` | 複製檔案進 Image |
| `RUN` | 建置時執行指令(例如 `dotnet publish`) |

> 基底 Image 與 Linux 的 Kernel / Userland(`FROM` 怎麼讀、Alpine vs Ubuntu、怎麼查有哪些 Image)見 [Base Image and Linux Userland.md](<Base Image and Linux Userland.md>)。

> **每道指令都會在前一層上疊一個新的 layer**
> ```
> ┌ layer 3:dotnet publish 的編譯結果
> ├ layer 2:COPY 進來的原始碼
> └ layer 1:基底 Image(OS + .NET)
> ```

> **Image 唯讀(immutable)、Container 有自己的可寫層**
> 同一個 Image 的多個 Container 共用唯讀的 Image;執行期的變動(log、上傳檔)存在各自的 writable layer(同樣分層)。
> ```
> Container 1 ─┐ 可寫層(各自獨立)
> Container 2 ─┤ 可寫層
>              └→ 共用唯讀的 Image(layer 1~3)
> ```

> 補充(我的理解,未驗證):老師說變動存在 Container 的 memory,是口語簡化;可寫層應是存在磁碟上,**隨容器一起刪除**。這解釋了為什麼 `docker rm` 後資料會不見,以及為什麼資料庫要掛 Volume(Volume 獨立於可寫層之外)。

### 六大好處
| 好處 | 說明 |
|---|---|
| 隔離 | 同一台主機同時跑 ASP.NET Core 8、.NET Framework 4、ASP.NET Core 5 的容器互不影響 |
| 可攜性 | Image 輕量,容易在機器間傳送 |
| 一致性 | 程式、相依、設定一起打包,環境差異不再造成意外錯誤 |
| 可擴展 | 流量增加就用同一個 Image 多開 Container |
| 部署簡單 | 裝 Docker Engine → 下載 Image → `docker run` |
| 版本管理(tag) | 1.0 / 1.1;新版有問題馬上切回舊版,並查 Container log |

老師提醒:要遵守最佳實務,注意安全與主機資源。

## 4. 關鍵概念

> **Image 含 .NET 執行環境**
> 打包 Products 服務成 Image 時,.NET 10 Runtime 放在 Image 裡,主機不用裝 .NET。傳統「測試機只有 OS」的問題,在這裡不存在。

> **`PackageReference` 版本**
> `Version="10.0.13"` 是「最低版本」(可能被升級);`[10.0.13]` 才是精確鎖定;`10.*` 是浮動。`packages.lock.json` 可記錄所有間接相依(需 `RestorePackagesWithLockFile`,我沒驗證過)。
> csproj 管不到:.NET SDK / Runtime 版本、作業系統與系統函式庫、資料庫版本、環境變數、Web 伺服器。

### Volume 與資料持久性(用 `products-mysql` 當例子)

> **資料不存在 API Container,存在 MySQL Container 掛載的 Volume。**
> API Container 是無狀態的,刪掉重開都沒差。

```
API Container ×N   ← 無狀態
      ↓ 連線
MySQL Container    ← 資料在它掛的 Volume(/var/lib/mysql)
```

| 類型 | 說明 |
|---|---|
| Bind mount | 自己指定主機資料夾,例 `-v /Users/lyx/data:/var/lib/mysql` |
| Named volume | Docker 管理,用名字引用,例 `-v products-mysql-data:/var/lib/mysql`(`ecommerce_pgdata` 就是) |
| Anonymous volume | 沒寫 `-v`,但 Image 有宣告 VOLUME 時自動建,名字是一串亂碼(`products-mysql` 就是) |

| 情境 | 結果 |
|---|---|
| `docker stop` / `start` | 資料還在 |
| `docker rm -f` 再 `docker run` 同名、沒指定同一個 volume | 建新的空 volume,**看不到舊資料**(資料跟 volume,不跟容器名稱) |
| 舊 volume | 變成孤兒,用 `-v <舊名稱>:/var/lib/mysql` 可救回 |
| `docker rm -v` / `docker volume prune` | 連 volume 一起刪,才是真的消失(我的理解,沒實際執行) |

> 之後 Docker 化 Products:`-v products-mysql-data:/var/lib/mysql`,用有名字的 volume。

## 5. 作業(自己練習)
1. 在某專案開啟 `RestorePackagesWithLockFile`,`dotnet restore` 後看 `packages.lock.json`
2. 同時開 2 個 mysql 容器(不同 port),用 `docker stats` 看 RAM,對照「不預先配給資源」
3. `docker volume ls`、`docker inspect <容器> --format '{{range .Mounts}}...{{end}}'` 看自己每個容器掛了什麼;把 `products-mysql` 改成有名字的 volume(**先備份,不要直接 `rm -f`**)
4. 列出 Products 服務在「別人的機器」上要跑起來需要的所有東西(SDK、MySQL、連線字串、`dotnet ef`…),當作 Dockerfile / compose 的清單

## 6. 下一步
已上完:VM vs Container 的技術差異、Image 與 Container 內部(分層檔案系統)。下一堂:Docker Image 實作(寫 Dockerfile) → 安裝 → Port mapping → Volumes → 把 UserService、Products Docker 化(Dockerfile + docker-compose)。
