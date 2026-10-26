# 2_13 全域例外處理 Middleware

> **一句話**:一個站在管線最前面的「總保全」,用 `try/catch` 包住後面所有環節,出事時記 log、回乾淨統一的 500,不把內部細節洩漏給前端。

## 1. 為什麼需要
沒有它,每個 Controller 方法都要自己寫 `try/catch`,程式很髒;而且未處理的例外預設會吐出 Stack Trace(含路徑、欄位名稱),是資安漏洞,使用者體驗也差。

## 2. 步驟

**① 建立 `eCommerce.API/Middlewares/ExceptionHandlingMiddleware.cs`**

```csharp
namespace eCommerce.API.Middlewares;

public class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;

    public ExceptionHandlingMiddleware(RequestDelegate next, ILogger<ExceptionHandlingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task Invoke(HttpContext httpContext)
    {
        try
        {
            await _next(httpContext);          // 放行給後面的環節
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "未預期錯誤,網址: {Path}", httpContext.Request.Path);
            await HandleExceptionAsync(httpContext, ex);
        }
    }

    private static Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        context.Response.ContentType = "application/json";
        context.Response.StatusCode = StatusCodes.Status500InternalServerError;
        return context.Response.WriteAsJsonAsync(new { error = "伺服器內部發生錯誤,請稍後再試。" });
    }
}

// 擴充方法:讓 Program.cs 可以寫 app.UseExceptionHandlingMiddleware()
public static class ExceptionHandlingMiddlewareExtensions
{
    public static IApplicationBuilder UseExceptionHandlingMiddleware(this IApplicationBuilder builder)
        => builder.UseMiddleware<ExceptionHandlingMiddleware>();
}
```

**② 在 `Program.cs` 註冊,放在 `Build()` 之後的第一行**

```csharp
var app = builder.Build();
app.UseExceptionHandlingMiddleware();   // ← 第一個
app.UseRouting();
app.MapControllers();
```

## 3. 關鍵概念

> **Middleware 順序 = 執行順序(俄羅斯娃娃)**
> 第一個註冊的最外層。它呼叫 `_next()` 把請求往內傳,後面所有環節(Routing、Controller、Service)都被包在它的 `try` 裡。放到最後面,前面關卡出的錯就接不到。

> **`await _next(httpContext)` = 「放行,但我盯著你」**
> 後面任何地方丟例外,都會彈回這裡的 `catch`。

| 它做的三件事 | 說明 |
|---|---|
| 萬能 `try/catch` | 不用每個方法自己包 |
| 保護內部資訊 | Stack Trace 只寫進 log,前端只看到固定的 JSON |
| 統一狀態碼 | 前端用一致邏輯處理 500 |

| 名詞 | 用途 |
|---|---|
| `RequestDelegate _next` | 管線的下一個環節,框架注入 |
| `ILogger<T>` | 日誌;泛型參數 T 成為 log 的分類名稱 |
| `UseMiddleware<T>()` | 把類別接上管線 |
| `this IApplicationBuilder` | 把方法「插」到 `app` 上的擴充方法語法 |

> ⚠️ 早期版本回應裡有 `detail = exception.Message`。例外訊息可能含連線字串、SQL、路徑,**不要回給 client**,只寫 log(Products 微服務的版本已移除)。

> 類比:Controller 打破玻璃杯(例外),門口總保全一邊無線電通知後台記錄(log),一邊對顧客說「內部出了點小狀況,請稍後再試」(統一 JSON)。
