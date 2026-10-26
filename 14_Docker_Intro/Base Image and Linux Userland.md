# 基底 Image 與 Linux 的 Kernel / Userland

> **一句話**:`FROM` 決定「裝了什麼軟體、底下是哪一套 Linux userland」;容器共用**主機的 kernel**,所以挑 Linux 種類,是在挑 userland,不是挑另一個 kernel。

## 1. 基底 Image(Base Image)

`FROM` 後面接的就是基底 Image,**從名稱和 tag 就決定了 .NET 版本與底層 userland**:

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:10.0          # Linux + ASP.NET 10 Runtime(只能執行,體積較小)
FROM mcr.microsoft.com/dotnet/sdk:10.0             # Linux + .NET 10 SDK(能 build / publish,體積較大)
FROM mcr.microsoft.com/dotnet/aspnet:10.0-alpine   # 同上,但用精簡的 Alpine Linux
```

| 部分 | 意思 |
|---|---|
| `mcr.microsoft.com/dotnet/aspnet` | Microsoft 的 Image 倉庫,ASP.NET Runtime 這個 Image |
| `:10.0` | **tag**,指定 .NET 版本;`8.0` 也存在(我查過 MCR 上有 `aspnet:10.0`、`10.0-alpine`、`8.0` 與 `sdk:10.0`) |
| `-alpine` / `-noble` 等後綴 | 指定底層 Linux **userland** 的種類(Alpine、Ubuntu Noble…) |
| `aspnet` vs `sdk` | Runtime 只跑得動程式;SDK 才能編譯。實務上 build 用 `sdk`、最後執行用 `aspnet`(是否分階段、怎麼寫,等老師講 Dockerfile) |

> **口訣**:Image 名稱 = 裝了什麼軟體(`aspnet` / `sdk`);tag 的版本 = 軟體版本;tag 的後綴 = 底下的 userland。

### 容易混淆的地方
`aspnet:10.0` 與 `aspnet:10.0-alpine` **都有** ASP.NET 10 Runtime,執行程式的能力一樣。差別只在 userland(體積、工具、C 函式庫)。不加後綴也有 userland,只是用預設的(是哪個發行版我沒查,可用 `docker run --rm mcr.microsoft.com/dotnet/aspnet:10.0 cat /etc/os-release` 確認)。

這就是為什麼「不用在主機裝 .NET」:.NET 在基底 Image 裡。要換 .NET 版本,改 tag 就好。

> 我的 UserService 的 `eCommerce.API/Dockerfile` 目前是**空檔案**(第 1 章 `touch` 建立,還沒寫內容)。

## 2. Linux 的 OS = Kernel + Userland

| 部分 | 是什麼 | 誰提供 |
|---|---|---|
| **Kernel** | 管硬體、記憶體、行程、檔案系統的核心 | **主機**(所有容器共用) |
| **Userland** | kernel 之外的一切:檔案系統結構、shell、C 函式庫、套件管理員(`apt`、`apk`)、基本工具 | **Image**(各容器自己帶) |

Ubuntu、Debian、Alpine 的差別幾乎都在 userland,底下用的都是同一種 Linux kernel。

```
主機(例如一台 Ubuntu VM)
└ Linux kernel(只有這一個,所有容器共用)
    ├ 容器 A:Alpine userland + .NET
    ├ 容器 B:Debian userland + .NET
    └ 容器 C:Ubuntu userland + .NET
```

> 所以「挑 Linux 種類」是在挑**容器要帶哪一套 userland**,不是挑另一個 kernel。主機是 Ubuntu 也能跑 Alpine 容器。

| | Alpine | Ubuntu / Debian 系 |
|---|---|---|
| 體積 | 很小 | 較大 |
| C 函式庫 | musl | glibc |
| 工具 | 精簡,內建少 | 較完整,好除錯 |
| 風險 | 少數依賴 glibc 的原生套件可能不相容 | 相容性好 |

(以上比較是我的知識,未在自己的專案驗證。)

## 3. 我的環境(查 `docker info`)

| 項目 | 結果 | 意思 |
|---|---|---|
| Mac CPU 架構(`uname -m`) | `x86_64` | 要用 amd64 / x86_64 的 Image |
| Docker 背後的 kernel | `5.10.25-linuxkit` | Docker Desktop 那台 Linux VM 的 kernel,容器共用它 |
| Docker 的 OS 類型 | `linux` | 容器是 Linux 容器 |

> 跑 `aspnet:10.0-alpine` 時:**kernel** = `5.10.25-linuxkit`(Docker Desktop 的 Linux VM),**userland** = Alpine。

## 4. 怎麼知道有哪些 Image 可選

| 方法 | 做法 |
|---|---|
| 看官方頁面 | MCR / Docker Hub 的說明頁,列出所有 tag 與支援的 OS、架構 |
| 查 tag 清單 | `curl -s https://mcr.microsoft.com/v2/dotnet/aspnet/tags/list` |
| 直接拉 | `docker pull mcr.microsoft.com/dotnet/aspnet:10.0-alpine`,不存在會報錯 |
| 看容器的 userland | `docker run --rm <image> cat /etc/os-release` |
| 看 Docker VM 的 kernel | `docker run --rm alpine uname -r` |

## 5. 相容性要注意的兩種情況
- **CPU 架構不符**:Image 要有對應的 amd64 / arm64。
- **主機 kernel 太舊**:缺少程式需要的功能。是否影響 .NET 10 的 Image,我沒驗證,實作 Dockerfile 時就會知道。

> 為什麼 Mac 要先開 Linux VM:容器要的是 **Linux kernel**,macOS 的 kernel 不是。VM 裡的 kernel 只有一個,上面的容器各自帶自己選的 userland。

## 6. 問題:`10.0` 換成 `10.0-alpine`,程式碼要改嗎?

> **問**:把 `FROM mcr.microsoft.com/dotnet/aspnet:10.0` 換成 `10.0-alpine`,Products 服務的程式碼要改嗎?要驗證什麼才知道沒問題?
>
> **我的答案**:理論上不用改。
>
> **補充**:對,C# 程式編譯成 IL,兩個 Image 都是同一個 ASP.NET 10 Runtime,換 userland 不影響程式碼。但「理論上」要靠驗證補上:換的是**底下的環境**,所以要確認程式有沒有依賴到環境。

| 要驗證 | 為什麼 |
|---|---|
| 容器能啟動 | 最基本,Runtime 與相依都載得起來 |
| 連得上資料庫 | `MySql.EntityFrameworkCore` 是否有原生函式庫相依,我不確定,實測最準 |
| 六個 endpoint 都正常 | 包含 FluentValidation、AutoMapper、Swagger |
| 文化、日期、貨幣格式 | Alpine 版預設可能少了國際化(ICU)相關函式庫,影響字串比較、日期格式(我的印象,細節未驗證,需要時查官方說明) |
| 時區 | 精簡的 userland 可能沒有完整時區資料 |
| 有沒有用原生套件(依賴 glibc 的) | Alpine 用 musl,少數原生套件不相容 |

> **原則**:程式碼沒變,不代表行為沒變。換基底 Image 就要重跑一次測試。

> 實作 Dockerfile 時,把 Products 服務分別用 `10.0` 與 `10.0-alpine` 各 build 一次,跑上面的驗證清單,看有沒有差異、Image 體積差多少。

## 7. 面試:你要用什麼 base image?

> **30 秒回答**:「我們是 .NET 10 的服務,用微軟官方的 Image。**build 階段用 `sdk:10.0`,最後執行用 `aspnet:10.0`**,因為執行只需要 Runtime,Image 小、攻擊面也小。版本**鎖定 tag**,不用 `latest`,各環境才一致。Linux 種類預設選相容性好的標準版;要壓縮體積會考慮 `-alpine`,但先實測原生相依與資料庫連線沒問題。」

| 要決定的 | 選擇 | 理由 |
|---|---|---|
| 要什麼軟體 | build:`sdk`;執行:`aspnet` | 執行不需要編譯器,小又安全 |
| .NET 版本 | 鎖定,例如 `10.0` | 與專案的 `net10.0` 一致;不用 `latest`(不可重現) |
| userland | 預設標準版;想省空間用 `-alpine` | Alpine 小但用 musl,少數原生套件可能不相容,換了要重跑驗證 |
| 來源 | 微軟官方 `mcr.microsoft.com/dotnet/...` | 有維護、有安全更新 |
| 架構 | 對應部署環境的 CPU(amd64 / arm64) | 不符就跑不起來 |

常被追問:
- **為什麼不用 Alpine,不是比較小嗎?** 小,但 musl 與 glibc 不同,原生套件可能出問題;先測,有明顯體積需求再換。
- **為什麼不用 `latest`?** 今天和三個月後拉到的版本可能不同,正是 Docker 要解決的「環境不一致」。

> 「分階段建置」「鎖 tag」「Alpine 的 musl 風險」是我的知識或一般實務,老師還沒講到,沒有在我的專案驗證。面試被問到沒做過的部分,直接說「原則是這樣,實際取捨依測試結果決定」。

### 問:最後的 Image 裡面還有 SDK 嗎?

> **我的答案**:沒有。

分階段建置(multi-stage build,我的知識,等老師講 Dockerfile 時核對):

```dockerfile
# 階段 1:build,用 SDK
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
COPY . .
RUN dotnet publish -c Release -o /app

# 階段 2:最終 Image,只用 Runtime
FROM mcr.microsoft.com/dotnet/aspnet:10.0
COPY --from=build /app /app      # 只把「編譯結果」拿過來
ENTRYPOINT ["dotnet", "/app/APILayer.dll"]
```

| 階段 | 基底 | 裡面有什麼 | 留在最終 Image 嗎 |
|---|---|---|---|
| build | `sdk:10.0` | SDK、原始碼、編譯過程 | **不會** |
| 最終 | `aspnet:10.0` | Runtime + 複製過來的編譯結果 | 就是出貨的 Image |

> 最終 Image 的 layer 從 `aspnet:10.0` 開始疊,**沒有包含 build 階段的 layer**,所以 SDK 與原始碼都不在裡面。這是分層檔案系統的應用:你可以選擇哪些 layer 進最後的 Image。

| | 單階段(全程用 `sdk`) | 分階段 |
|---|---|---|
| 最終 Image 含 SDK、原始碼 | 有 | **沒有** |
| 體積 | 大 | 小 |
| 安全 | 攻擊面大 | 攻擊面小,只剩執行所需 |
