# 6_17 AuthController:註冊與登入端點

> **一句話**:Controller 只做三件事:收 HTTP 請求、呼叫 Service、依結果回對的狀態碼。

## 1. 為什麼需要
把 Service 變成 HTTP API,前端(Angular / Postman)才能呼叫。

## 2. 步驟:`API/Controllers/AuthController.cs`

```csharp
[Route("api/[controller]")]   // 控制器名 AuthController → api/auth
[ApiController]
public class AuthController : ControllerBase
{
    private readonly IUsersService _usersService;

    public AuthController(IUsersService usersService)   // 建構子注入
    {
        _usersService = usersService;
    }

    [HttpPost("register")]
    public async Task<IActionResult> Register(RegisterRequest registerRequest)
    {
        if (registerRequest == null) return BadRequest("Invalid registration data");

        AuthenticationResponse? response = await _usersService.Register(registerRequest);

        if (response == null || response.Success == false) return BadRequest(response);
        return Ok(response);
    }

    [HttpPost("login")]
    public async Task<IActionResult> Login(LoginRequest loginRequest)
    {
        if (loginRequest == null) return BadRequest("Invalid login data");

        AuthenticationResponse? response = await _usersService.Login(loginRequest);

        if (response == null || response.Success == false) return Unauthorized(response);
        return Ok(response);
    }
}
```

## 3. 關鍵概念

| 狀態碼 | 回傳方法 | 使用時機 |
|---|---|---|
| 200 | `Ok(response)` | 成功 |
| 400 | `BadRequest(...)` | 請求資料有問題(註冊失敗) |
| 401 | `Unauthorized(...)` | 登入失敗 |

> **登入為什麼用 POST,不能用 GET**
> GET 的參數會在網址上,被記進瀏覽器歷史與伺服器日誌,帳密不能外洩。POST 的資料放 Body。

| Attribute | 作用 |
|---|---|
| `[ApiController]` | 宣告這是 Web API 控制器,含自動模型驗證等行為 |
| `[Route("api/[controller]")]` | 路由前綴,`[controller]` 取類別名去掉 `Controller` |
| `[HttpPost("register")]` | 此方法處理 `POST api/auth/register` |

## 4. 下一步
用 Postman 測試這兩個端點 → 見 7_18。
