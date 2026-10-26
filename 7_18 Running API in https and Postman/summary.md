# 7_18 用 HTTPS 執行 API,Postman 測試與 VS Code 除錯

> **一句話**:用 `https` profile 啟動 API,用 Postman 打 `register` / `login`,在 VS Code 用 F5 加中斷點追流程;順便碰到並解決了 enum 綁定問題。

## 1. 為什麼需要
端點寫好了要驗證。Postman 是最普及的 API 測試工具,中斷點能看到資料怎麼從 Controller 流到 Repository。

## 2. 步驟

### ① 啟動 API(HTTPS)
```bash
dotnet dev-certs https --trust          # 只需一次,macOS 會問密碼
cd eCommerce.API
dotnet run --launch-profile https       # 預設 dotnet run 用第一個 profile(http)
```
監聽 `https://localhost:7032` 與 `http://localhost:5291`。

> 憑證過期:先 `dotnet dev-certs https --clean` 再重新 `--trust`。
> 沒信任憑證時,`curl` 可加 `-k` 略過檢查。
> `Program.cs` 沒呼叫 `UseHttpsRedirection()` 就不會把 HTTP 轉到 HTTPS,無害,直接打 HTTPS 網址即可。
> ⚠️ **在 Docker 裡跑時,開發憑證不存在**:要掛載憑證並設 `ASPNETCORE_Kestrel__Certificates__Default__Path` 與 `...__Password`(Docker 章節會用到)。

### ② Postman 測試
| 項目 | 設定 |
|---|---|
| Login | `POST https://localhost:7032/api/auth/login` |
| Register | `POST https://localhost:7032/api/auth/register` |
| Body | `raw` → `JSON`(POST 的資料放 Body,GET 的參數放 Params) |

### ③ 遇到的錯誤與解法
| 錯誤 | 原因 | 解法 |
|---|---|---|
| `unable to verify the first certificate` | Postman 不信任本機自簽憑證 | Postman Settings → General 關閉 SSL certificate verification |
| `415 Unsupported Media Type` | 沒帶 JSON body | 送出 JSON,Body 選 `raw` → `JSON` |
| 驗證失敗,JSON 值無法轉成 `gender` | ASP.NET Core 預設不把字串轉成 enum | `AddControllers().AddJsonOptions(o => o.JsonSerializerOptions.Converters.Add(new JsonStringEnumConverter()))` |

> enum 轉換要設在 **controller 層級**:model binding 發生在 controller 執行之前。

### ④ 沒驗證時的現象(當時的限制)
- 送空 JSON 照樣回 200,email、password 是 `null`,因為還沒有模型驗證(之後用 FluentValidation,見 10)
- Repository 是假的:`AddUser` 只產生 Guid,沒存資料庫(之後見 9)

## 3. VS Code(macOS)用 F5 除錯

### 事前準備
- 安裝 **C# Dev Kit** 或 **C#** 擴充套件
- `dotnet dev-certs https --trust`

### `.vscode/tasks.json`:除錯前先 build
```json
{
  "version": "2.0.0",
  "tasks": [{
    "label": "build",
    "command": "/usr/local/bin/dotnet",
    "type": "process",
    "args": ["build", "${workspaceFolder}/eCommerce.API/eCommerce.API.csproj",
             "/property:GenerateFullPaths=true", "/consoleloggerparameters:NoSummary"],
    "problemMatcher": "$msCompile"
  }]
}
```

### `.vscode/launch.json`:啟動 HTTPS 設定
```json
{
  "version": "0.2.0",
  "configurations": [{
    "name": "eCommerce.API (https)",
    "type": "coreclr",
    "request": "launch",
    "preLaunchTask": "build",
    "program": "${workspaceFolder}/eCommerce.API/bin/Debug/net10.0/eCommerce.API.dll",
    "cwd": "${workspaceFolder}/eCommerce.API",
    "stopAtEntry": false,
    "env": {
      "ASPNETCORE_ENVIRONMENT": "Development",
      "ASPNETCORE_URLS": "https://localhost:7032;http://localhost:5291"
    }
  }]
}
```

### 除錯步驟
1. 在程式碼行號左邊點一下設**中斷點**(例如 `AuthController` 的 `Login`)
2. 按 **F5**,Debug Console 出現 `Now listening on: https://localhost:7032`
3. Postman 送請求,程式停在中斷點
4. 左側 **Variables** 面板看變數,或滑鼠移到變數上

| 動作 | 快捷鍵 |
|---|---|
| Continue | `F5` |
| Step Over(不進方法) | `F10` |
| Step Into(進入方法) | `F11` |
| Step Out | `Shift+F11` |
| Restart | `Cmd+Shift+F5` |
| Stop | `Shift+F5` |

> MacBook 的 F 鍵如果預設是亮度 / 音量,要加 `Fn`。

### 踩過的坑
| 錯誤 | 原因 | 解法 |
|---|---|---|
| build 找不到檔案 | 路徑多寫了一層資料夾,但 `${workspaceFolder}` 已是該層 | 路徑改成 `${workspaceFolder}/eCommerce.API/...` |
| `Path to shell executable "dotnet" does not exist` | 從 Dock 開 VS Code,PATH 裡沒有 dotnet | `command` 改成絕對路徑 `/usr/local/bin/dotnet`,或用終端機 `code .` 開啟 |

## 4. 關鍵概念

> **VS Code 與 Visual Studio 的差異**
> VS Code 不會自動開瀏覽器,直接用 Postman 測。停止除錯後 API 就停了,Postman 連不到。
> 用 `dotnet run --launch-profile https` 的終端機模式**不能**用中斷點,要除錯必須 F5。

> **看追蹤流程學到的**
> Controller → `UserService.Login` → `Repository.GetUserByEmailAndPassword`。早期 Repository 直接 new 一個假使用者,Service 看到不是 `null` 就回 `Success = true`。

## 5. 下一步
導入 AutoMapper 簡化物件對應 → 見 8_19_20。

> 截圖:資料夾內有兩張 Postman 的截圖(`Screenshot 2026-09-20 ...png`)。
