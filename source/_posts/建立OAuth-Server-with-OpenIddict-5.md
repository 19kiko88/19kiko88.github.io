---
title: 建立OAuth Server with OpenIddict(5)
date: 2026-11-09 14:17:50
categories:
  - 後端開發
tags:
  - OAuth
  - OpenIddict
---
##### 接續上一篇...
##### 此篇主要著重在資安補強的部分
- 把開發用憑證換成固定的簽章和加密憑證
- 加上全域的 null byte 攔截 middleware
- 所有回應加上安全標頭，並移除 Server 標頭
- 未處理的例外不洩漏技術細節
<!-- more -->
#
#
#
---
## 把開發用憑證換成固定的簽章和加密憑證（設計文件第 9.1、10 節）

這一步在做什麼：Program.cs 目前用的是：
options.AddDevelopmentEncryptionCertificate()
       .AddDevelopmentSigningCertificate();

OpenIddict 會在你這台電腦的 Windows 憑證存放區自動產生這兩張憑證。在你的電腦上它們是固定的，所以之前 kid 都沒變，但它有三個問題：
1. 換一台主機就不一樣：部署到正式機或多台主機時，每台各自產生一張，A 主機簽的 JWT，B 主機發布的 JWKS 裡就找不到公鑰。
2. 無法備份和輪替：設計文件第 9.1 節要求私鑰要有備援、版本控管，輪替時採「重疊式」，但開發憑證都辦不到。
3. OpenIddict 官方說明它只能用在開發環境。

這次改成：自己產生兩張憑證檔（.pfx），放在 repo 以外的地方，由設定指定路徑，密碼放在 user-secrets。將來正式環境只要把路徑和密碼改成指向金鑰管理機制提供的憑證，程式碼不用改。

兩張憑證的用途：
{% asset_image 憑證類別.jpg 憑證類別%}

- **簽章憑證**：私鑰簽章，公鑰驗證。<span style="color: red;">證明資料來源(防止資料被竄改)，主要確認身分來源跟資料的完整性。</span>
    應用場景(RS256)如下：以JWT傳輸為例
    1. OAuth Server： 使用自己的私鑰對 JWT 產生數位簽章。
    2. MCP Server： 從 OAuth Server 公開的 JWKS Endpoint（例如 /.well-known/jwks.json）取得公鑰。
    3. 驗證 JWT： MCP Server 使用公鑰驗證 JWT 的數位簽章，確認簽章有效且內容未遭竄改。

    <span style="color: red;">簽章憑證的重點不在資料加密，而在於驗證資料的完整性與簽發來源。</span>


- **加密憑證**：公鑰加密，私鑰解密。<span style="color: red;">保護資料不被偷看(防止資料被偷看)，主要確認資料的機密性。</span>
    應用場景(RSA)如下：假設 MCP Server 要傳送員工資料給 HR Server。
    1. HR Server 建立一組 RSA 金鑰。
    2. HR Server 將公鑰憑證提供給 MCP Server。
    3. MCP Server 使用 HR Server 的公鑰加密薪資資料。
    4. MCP Server 將密文傳送給 HR Server。
    5. HR Server 使用自己的私鑰解密，取得原始薪資資料。    

    <span style="color: red;">加密憑證（Encryption Certificate）的重點不在驗證資料來源，而在於保護資料的機密性，避免未經授權的人讀取資料內容。</span>

看完簽章憑證跟加密憑證的差異後，開始進行把開發用憑證換成固定的簽章和加密憑證的流程


1. 產生兩張憑證檔

先建立一個 repo 以外的資料夾，例如 D:\Homer\Secrets，然後在 PowerShell 執行。密碼請換成你自己的，兩張要記住各自的密碼：
``` bash
$dir = 'D:\Homer\Secrets'

# 簽章憑證（KeyUsage = DigitalSignature）
$cert = New-SelfSignedCertificate -Subject 'CN=OAuth AuthorizationServer Signing' -KeyAlgorithm RSA -KeyLength 2048 -KeyUsage DigitalSignature -KeyExportPolicy Exportable -CertStoreLocation 'Cert:\CurrentUser\My' -NotAfter (Get-Date).AddYears(2)
Export-PfxCertificate -Cert $cert -FilePath "$dir\oauth-signing.pfx" -Password (ConvertTo-SecureString '換成簽章憑證密碼' -AsPlainText -Force)
Remove-Item "Cert:\CurrentUser\My\$($cert.Thumbprint)"

# 加密憑證（KeyUsage = KeyEncipherment）
$cert = New-SelfSignedCertificate -Subject 'CN=OAuth AuthorizationServer Encryption' -KeyAlgorithm RSA -KeyLength 2048 -KeyUsage KeyEncipherment -KeyExportPolicy Exportable -CertStoreLocation 'Cert:\CurrentUser\My' -NotAfter (Get-Date).AddYears(2)
Export-PfxCertificate -Cert $cert -FilePath "$dir\oauth-encryption.pfx" -Password (ConvertTo-SecureString '換成加密憑證密碼' -AsPlainText -Force)
Remove-Item "Cert:\CurrentUser\My\$($cert.Thumbprint)"
```
{% asset_image pfx_created.jpg pfx_created%}

2. 設定路徑和密碼
OAuth.AuthorizationServer/appsettings.Development.json 加上路徑（路徑不是機密，可以放設定檔）：
``` json
{
  "Certificates": {
    "Signing": { "Path": "D:\\Homer\\Secrets\\oauth-signing.pfx" },
    "Encryption": { "Path": "D:\\Homer\\Secrets\\oauth-encryption.pfx" }
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Trace"
    }
  }
}
```

密碼放 user-secrets（在 D:\Homer\Project\OAuth 下執行）：
``` bash
dotnet user-secrets set "Certificates:Signing:Password" "換成簽章憑證密碼" --project OAuth.AuthorizationServer
dotnet user-secrets set "Certificates:Encryption:Password" "換成加密憑證密碼" --project OAuth.AuthorizationServer
```
<span style="color: red;">這邊的密碼跟建立憑證時的密碼是一樣的</span>

3. Program.cs：改用這兩張憑證

最上面加上：
``` csharp
using System.Security.Cryptography.X509Certificates;
```

把原本這段：
``` csharp
// 開發階段用的簽章/加密憑證：之後正式環境一定要換成
// 真正保管好的簽章憑證（見設計文件第9節注意事項2、第8.1節）
options.AddDevelopmentEncryptionCertificate()
       .AddDevelopmentSigningCertificate();
```

換成：
``` csharp
// 固定的簽章/加密憑證（設計文件第9.1、10節）：路徑由設定指定、密碼放在 user-secrets（正式環境改由金鑰管理機制提供）。
// 簽章憑證：簽 JWT（RS256），公鑰發布在 JWKS；加密憑證：加密 Authorization Code、refresh_token
options.AddSigningCertificate(LoadCertificate(builder.Configuration, "Certificates:Signing"))
       .AddEncryptionCertificate(LoadCertificate(builder.Configuration, "Certificates:Encryption"));
```

在檔案最下面、SeedDataAsync 的後面新增：
``` csharp
static X509Certificate2 LoadCertificate(IConfiguration configuration, string section)
{
    var path = configuration[$"{section}:Path"]
        ?? throw new InvalidOperationException($"找不到設定 '{section}:Path'");
    var password = configuration[$"{section}:Password"]
        ?? throw new InvalidOperationException($"找不到設定 '{section}:Password'");

    return X509CertificateLoader.LoadPkcs12FromFile(path, password);
}
```

幾個重點說明：

- 為什麼 KeyUsage 要分開設：憑證的 KeyUsage 規定這把金鑰「可以做什麼」。OpenIddict 會檢查：簽章憑證必須有 DigitalSignature，加密憑證必須有 KeyEncipherment，不符合就啟動失敗。
- 為什麼產生後要從憑證存放區刪掉：New-SelfSignedCertificate 一定要先把憑證建在 Windows 憑證存放區裡，匯出成 .pfx 後就不需要了。刪掉可以確保程式真的是從檔案讀取，不是不小心從存放區讀到。
- .pfx 為什麼要放在 repo 外面：它包含私鑰。私鑰外洩的話，任何人都能簽出「看起來是我們發的」JWT。放在 repo 外面，就不會被 commit 進 git。
- X509CertificateLoader.LoadPkcs12FromFile：.NET 9 之後建議用的載入方式。舊的 new X509Certificate2(path, password) 已經被標記為過時。
- 有效期 2 年：到期前要輪替。設計文件第 9.1 節的重疊式輪替（先發布新公鑰，再切換簽發用的私鑰），之後要用時再實作。

⚠️ 換憑證之後，舊的 token 全部失效
- 舊的 access_token：用舊私鑰簽的，JWKS 裡已經沒有對應的公鑰。
- 舊的 Authorization Code、refresh_token：用舊憑證加密的，新憑證解不開。

開發環境沒關係，重跑一次流程就好。但這也說明了為什麼正式環境一定要做重疊式輪替。

測試時要注意：只動了 Authorization Server，MockPnp 不用改。

1. 重啟 Authorization Server，開啟 https://localhost:7280/.well-known/jwks，記下 kid。
2. 再重啟一次，再看一次 kid。
3. 跑一次完整流程，加上 refresh。

請執行後確認：
1. dotnet build 正常編譯，Server 正常啟動，沒有憑證相關的例外。
2. 第 1 步的 kid 和你之前看到的不同，代表換成了新的簽章憑證。
3. 第 2 步重啟後 kid 沒有變，代表憑證是固定的。
4. 第 3 步完整流程和 refresh 都正常。access_token Header 裡的 kid 和 JWKS 一致。

.well-known/jwks裡的kid
{% asset_image kid_check_1.jpg kid_check_1 %}

acess_token解碼出的jwt header。kid跟.well-known/jwks裡的kid一樣。
{% asset_image kid_check_2.jpg kid_check_2 %}

refresh_token解碼出的jwt header。kid跟.well-known/jwks裡的kid也是一樣。
{% asset_image kid_check_3.jpg kid_check_3 %}


**到這個步驟為止：改用我們自己產生、自己管理的固定憑證，取代 OpenIddict 自動產生的開發憑證。**

{% asset_image 憑證比較.jpg 憑證比較 %}

另外說明一點：<span style="color: red;">JWT 簽章用自簽憑證是正常的，正式環境也不需要向 CA 申請。</span>Resource Server 是靠 JWKS 公鑰驗簽，信任來自「JWKS 是從我們的 Authorization Server 拿的」，不是靠 CA 憑證鏈。這和網站 HTTPS 用的憑證不一樣，HTTPS 憑證才需要 CA 簽發。
#
#
#
---
## 加上全域的 null byte 攔截 middleware

這一步在做什麼：之前只在 bridge-callback 檢查了 null byte。但 SECURITY.md 第 1 節要求的是全域檢查：每個請求的 QueryString 和 Form 欄位只要出現 \0，就直接回 400。這次寫一個 middleware，放在 pipeline 最前面，所有端點都會經過它，包括 OpenIddict 的 /connect/authorize、/connect/token。

1. 新增檔案 OAuth.AuthorizationServer/Middleware/RejectNullByteMiddleware.cs
``` csharp
using Microsoft.Extensions.Primitives;

namespace OAuth.AuthorizationServer.Middleware;

/// <summary>
/// 比照 PCBRepairNavi SECURITY.md 第1節：QueryString 或 Form ）時，直接回 400
/// </summary>
public class RejectNullByteMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RejectNullByteMiddleware> _logger;

    public RejectNullByteMiddleware(RequestDelegate next, ILogger<RejectNullByteMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var request = context.Request;

        if (ContainsNullByte(request.Query)
            || (request.HasFormContentType && ContainsNullByte(await request.ReadFormAsync())))
        {
            _logger.LogWarning("Request rejected. Path={Path}, Reason=NullByte.", request.Path);
            context.Response.StatusCode = StatusCodes.Status400BadRequest;
            return;
        }

        await _next(context);
    }

    private static bool ContainsNullByte(IEnumerable<KeyValuePair<string, StringValues>> fields)
    {
        return fields.Any(f => f.Key.Contains('\0') || f.Value.Any(v => v is not null && v.Contains('\0')));
    }
}
```

2. Program.cs：註冊 middleware，並明確呼叫 UseAuthentication()

最上面加上：
``` csharp
using OAuth.AuthorizationServer.Middleware;
```

把原本的：
``` csharp
app.UseHttpsRedirection();

app.UseAuthorization();
```

改成：
``` csharp
// 最前面先擋掉含 null byte 的請求（SECURITY.md 第1節），之後的 OpenIddict、Controller 都不會收到
app.UseMiddleware<RejectNullByteMiddleware>();

app.UseHttpsRedirection();

// 明確呼叫，讓 OpenIddict 排在 null byte 檢查之後才處理請求
app.UseAuthentication();
app.UseAuthorization();
```

幾個重點說明：

- middleware 是什麼：每個 HTTP 請求進來後，會依序經過一連串的處理元件（pipeline），最後才到 Controller。middleware 就是其中一個元件，可以檢查請求、決定要不要往下傳。這裡沒有呼叫 _next(context) 就直接 return，請求就到此為止，後面的元件都不會執行。
- 為什麼要明確加上 app.UseAuthentication()：這是這一步最關鍵的地方。原本的程式沒有寫這行，但 OpenIddict 還是能運作，因為 .NET 偵測到有註冊認證服務時，會自動把 UseAuthentication() 加在 pipeline 的最前面，排在你寫的所有 middleware 之前。OpenIddict 就是掛在這個認證 middleware 裡處理 /connect/* 的。如果不明確寫出來，順序會是：
（自動加的）UseAuthentication → OpenIddict 先處理  ← null byte 還沒檢查
RejectNullByteMiddleware                         ← 太晚了
  明確寫出來，.NET 就不會自動加，順序才會照你寫的：先檢查 null byte，再交給 OpenIddict。
- request.ReadFormAsync() 讀過之後，OpenIddict 還能讀嗎：可以。ASP.NET Core 讀取 Form 後會快取起來，之後 OpenIddict 再讀，拿到的是同一份資料。
- 欄位名稱也要檢查：攻擊者也可能把 \0 放在參數名稱裡，例如 client%00_id=...，所以 Key 和 Value 都要檢查。
- bridge-callback 原本的 IsValidCallbackParameter 要不要拿掉：保留。它還檢查了必填和長度，而且多一層防護也沒有壞處。

測試時要注意：Insomnia 的 Form 欄位會把你輸入的 % 自動編碼成 %25，送出去會變成字面上的 %00 三個字元，不是 null byte。所以 POST 那項改用 curl.exe，它會原封不動送出。

1. QueryString 含 null byte：用 GET 送出（Follow redirects 開關都可以）：
https://localhost:7280/connect/authorize?client_id=pnp-mcp-client%00&response_type=code
2. Form 含 null byte：
curl.exe -k -s -o NUL -w "%{http_code}\n" -X POST https://localhost:7280/connect/token --data-raw "grant_type=refresh_token&refresh_token=abc%00def&client_id=pnp-mcp-client"
3. 正常流程：跑一次完整流程，換到 token。

請執行後確認：
1. dotnet build 正常編譯。
2. 第 1 項和第 2 項都回 400，Authorization Server 的 Console 各有一行 Request rejected. Path=..., Reason=NullByte.，而且沒有 OpenIddict 的錯誤訊息。這就證明請求在進到 OpenIddict 之前就被擋下了。
{% asset_image null_byte_test.jpg null_byte_test%}
3. 第 3 項正常換到 token。


補充：

- **觸發自動加入的條件**
用 WebApplication.CreateBuilder 建立的專案（.NET 7 以後），在 builder.Build() 時會檢查：
1. DI 裡有註冊認證服務（IAuthenticationSchemeProvider），而且你沒有自己呼叫 app.UseAuthentication() → 自動加上 UseAuthentication()。
2. DI 裡有註冊授權服務（IAuthorizationHandlerProvider），而且你沒有自己呼叫 app.UseAuthorization() → 自動加上 UseAuthorization()。
自動加入的位置在 pipeline 的最前面，排在你自己寫的 middleware 之前。

- **app.UseAuthentication()**：確認「你是誰」
它會在 pipeline 裡加入認證 middleware，在每個請求進來時，呼叫已註冊的認證處理器，從請求中辨識身分。

在這個專案裡，它有兩個作用：
1. 讓 OpenIddict Server 運作：OpenIddict 是掛在這個 middleware 裡的認證處理器。請求路徑是 /connect/authorize、/connect/token 時，由它先解析和驗證 OAuth 參數，再交給我們的 Controller。也因為這樣，Controller 裡才能用 HttpContext.GetOpenIddictServerRequest() 拿到請求。
2. 讓 OpenIddict Validation 運作：Program.cs 裡的 .AddValidation(...)，讓這個 App 可以驗證自己發出的 access_token，例如請求帶 Authorization: Bearer <JWT> 時。

- **app.UseAuthorization()**：確認「你可以做什麼」
它會檢查 Controller 或 action 上的 [Authorize] 屬性，判斷目前的身分有沒有權限執行。例如某個 action 標了 [Authorize]，沒有帶有效 token 的請求就會被拒絕。

在這個專案裡，目前沒有任何地方用到 [Authorize]，所以它實際上沒有作用。保留它是 ASP.NET Core 的慣例寫法，將來有端點需要保護時可以直接用，不影響現在的功能。

順序一定是先 Authentication，再 Authorization

UseAuthentication   → 先確認你是誰
UseAuthorization    → 再根據你是誰，判斷能不能存取

反過來的話，Authorization 檢查時還不知道你是誰，所有需要權限的請求都會被拒絕。

這一步為什麼要明確寫出 UseAuthentication()

原本不寫也能運作，是因為 .NET 偵測到有註冊認證服務時，會自動把它加在 pipeline 最前面。但這樣它就排在 RejectNullByteMiddleware 之前，OpenIddict 會先處理到含 null byte 的請求。明確寫在 UseMiddleware<RejectNullByteMiddleware>() 後面，才能確保先擋 null byte，再交給 OpenIddict。
#
#
#
---
## 所有回應加上安全標頭，並移除 Server 標頭（SECURITY.md 第 6、8、11、14 節）

下一步：所有回應加上安全標頭，並移除 Server 標頭（SECURITY.md 第 6、8、11、14 節）

這一步在做什麼：資安掃描工具（AppScan、Checkmarx）會檢查每個回應有沒有帶特定的安全標頭，缺少的話就會列為弱點。PCBRepairNavi 在 HeaderFilter 和 Global.asax 裡做了這件事，Authorization Server 也要比照。這次加一個 middleware，讓所有回應都帶上這些標頭，包括 OpenIddict 回的 JSON、302 導轉、被 null byte 擋下的 400。

{% asset_image remove_server_header.jpg remove_server_header%}

1. 新增檔案 OAuth.AuthorizationServer/Middleware/SecurityHeadersMiddleware.cs
``` csharp
namespace OAuth.AuthorizationServer.Middleware;

/// <summary>
/// 比照 PCBRepairNavi SECURITY.md 第6、11節：所有回應加上安全標頭。
/// Authorization Server 只回 JSON 與 302 導轉、不輸出 HTML，所以 CSP 採用最嚴格的 'none'
/// </summary>
public class SecurityHeadersMiddleware
{
    private readonly RequestDelegate _next;

    public SecurityHeadersMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public Task InvokeAsync(HttpContext context)
    {
        // 在交給後面的 middleware 之前就先設定，後面不論在哪裡產生回應（OpenIddict、Controller、被擋下的400），都會帶著這些標頭
        var headers = context.Response.Headers;
        headers["Content-Security-Policy"] = "default-src 'none'; frame-ancestors 'none'; base-uri 'none'; form-action 'none'";
        headers["Referrer-Policy"] = "no-referrer";
        headers["X-Content-Type-Options"] = "nosniff";

        return _next(context);
    }
}
```

2. Program.cs

在 var builder = WebApplication.CreateBuilder(args); 下面加上：
``` csharp
// 不在回應中透露伺服器軟體（SECURITY.md 第8節）
builder.WebHost.ConfigureKestrel(options => options.AddServerHeader = false);

// HSTS：1年、含子網域，不採用 preload（SECURITY.md 第14.3節）
builder.Services.AddHsts(options =>
{
    options.MaxAge = TimeSpan.FromDays(365);
    options.IncludeSubDomains = true;
});
```

pipeline 的部分改成：
``` csharp
// 所有回應加上安全標頭，放在最前面，連後面被擋下的請求也會帶著標頭
app.UseMiddleware<SecurityHeadersMiddleware>();

// HSTS 只在非開發環境啟用，避免瀏覽器把 localhost 記成「只能用 HTTPS」，影響本機其他開發中的網站
if (!app.Environment.IsDevelopment())
{
    app.UseHsts();
}

// 最前面先擋掉含 null byte 的請求（SECURITY.md 第1節），之後的 OpenIddict、Controller 都不會收到
app.UseMiddleware<RejectNullByteMiddleware>();

app.UseHttpsRedirection();

// 明確呼叫，讓 OpenIddict 排在 null byte 檢查之後才處理請求
app.UseAuthentication();
app.UseAuthorization();
```

幾個重點說明：

- 為什麼 CSP 和 SECURITY.md 的標準值不同：SECURITY.md 的 CSP 是給 PCBRepairNavi 這種會輸出 HTML 頁面的網站用的，要允許載入自己的 script、圖片、字型。Authorization Server 只回 JSON 和 302，瀏覽器根本不需要載入任何資源，所以全部設成 'none' 最安全。frame-ancestors、base-uri、form-action 不會繼承 default-src，所以要另外明確寫出來，這點 SECURITY.md 第 6.1 節也有提到。
- 為什麼 Referrer-Policy 用 no-referrer，而不是 SECURITY.md 的 strict-origin-when-cross-origin：SECURITY.md 不選 no-referrer 的理由，是 PCBRepairNavi 有 IsAllowedReferer() 要靠 Referer 做白名單驗證。Authorization Server 沒有這個需求，而我們的網址上帶著 nonce、bridgeCode、Authorization Code 這些敏感值，完全不送 Referer 最安全。
- HSTS 為什麼開發環境不加：瀏覽器收到 HSTS 後，會「記住」這個主機一年內只能用 HTTPS。如果對 localhost 加上 HSTS，你本機其他用 HTTP 跑的開發網站也會被強制改成 HTTPS，造成麻煩。ASP.NET Core 的 UseHsts() 本身也會自動略過 localhost。所以這個標頭在本機測不到，是正常的。
- AddServerHeader = false 的限制：它只移除 Kestrel 自己加的 Server: Kestrel。如果正式環境放在 IIS 後面，IIS 會再加上自己的 Server 標頭，要在 IIS 那邊另外設定。
- 中介軟體的順序：SecurityHeadersMiddleware 要排在 RejectNullByteMiddleware 前面，被 null byte 擋下的 400 回應才會帶到安全標頭。掃描工具常常就是故意送奇怪的請求，再檢查錯誤回應有沒有缺標頭。

測試時要注意：用 curl 看回應標頭比較清楚。-D - 會把回應標頭印出來。

1. 一般回應：
curl.exe -k -s -D - -o NUL https://localhost:7280/.well-known/openid-configuration
2. 被 null byte 擋下的回應：
curl.exe -k -s -D - -o NUL "https://localhost:7280/connect/authorize?client_id=abc%00"
3. 正常流程：跑一次完整流程，換到 token。

請執行後確認：
1. dotnet build 正常編譯。
2. 第 1、2 項的回應標頭都有 Content-Security-Policy、Referrer-Policy: no-referrer、X-Content-Type-Options: nosniff，而且沒有 Server: Kestrel。
{% asset_image header_compare.jpg header_compare %}
3. 沒有 Strict-Transport-Security，這是預期的，因為現在是開發環境。<span style="color: red;">(測試機沒有app.UseHsts();所以header不會有測試機沒有Strict-Transport-Security。)</span>
4. 第 3 項正常流程不受影響。
#
#
#
---
## 未處理的例外不洩漏技術細節（SECURITY.md 第 2.3 節）

這一步在做什麼：程式裡如果發生沒有被 catch 的例外，例如 IdentityProviderClient 發現設定錯誤而丟出 InvalidOperationException，目前的行為是：
- 開發環境：.NET 自動顯示「開發人員例外頁面」，完整列出例外訊息和 stack trace。開發時很方便。
- 非開發環境：回 500，body 是空的。沒有洩漏細節，但呼叫端完全不知道發生什麼事，維運人員也很難從大量 log 裡找出是哪一次請求出錯。

這次在非開發環境加上統一的錯誤處理：
- 回應只給通用訊息和追蹤編號（trace_id），不透露例外內容。
- 完整的例外記在 log 裡，也帶上同一個 trace_id。

使用者或 MCP Client 回報問題時，只要提供 trace_id，維運人員就能直接在 log 裡找到那一筆。

同時要修正上一步的 SecurityHeadersMiddleware：例外處理器在產生錯誤回應前，會清空所有已經設定的回應標頭，所以 500 回應會遺失安全標頭。要改成在「回應即將送出」那一刻才加標頭。

1. SecurityHeadersMiddleware.cs：InvokeAsync 改成用 OnStarting
``` csharp
public Task InvokeAsync(HttpContext context)
{
    // 用 OnStarting 在「回應即將送出」時才加上標頭：
    // 例外處理器產生500回應前會清空已設定的標頭，若在這裡直接設定，500回應就會缺少安全標頭
    context.Response.OnStarting(() =>
    {
        var headers = context.Response.Headers;
        headers["Content-Security-Policy"] = "default-src 'none'; frame-ancestors 'none'; base-uri 'none'; form-action 'none'";
        headers["Referrer-Policy"] = "no-referrer";
        headers["X-Content-Type-Options"] = "nosniff";
        return Task.CompletedTask;
    });

    return _next(context);
}
```

2. Program.cs：在 pipeline 的最前面加上例外處理

最上面加上：
``` csharp
using Microsoft.AspNetCore.Diagnostics;
```

在 var app = builder.Build(); 之後、app.UseMiddleware<SecurityHeadersMiddleware>(); 之前加上：
``` csharp
// 未處理的例外：回應只給通用訊息與追蹤編號，完整例外只寫進 log（SECURITY.md 第2.3節）。
// 開發環境保留 .NET 內建的開發人員例外頁面，方便除錯
if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler(errorApp => errorApp.Run(async context =>
    {
        var exception = context.Features.Get<IExceptionHandlerFeature>()?.Error;
        app.Logger.LogError(exception, "Unhandled exception. TraceId={TraceId}, Path={Path}.",
            context.TraceIdentifier, context.Request.Path);

        context.Response.StatusCode = StatusCodes.Status500InternalServerError;
        await context.Response.WriteAsJsonAsync(new
        {
            error = "server_error",
            error_description = "伺服器發生錯誤，請稍後再試。",
            trace_id = context.TraceIdentifier,
        });
    }));
}
```

幾個重點說明：

- 為什麼一定要放在 pipeline 最前面：例外會從發生的地方往外傳，只有排在它前面的 middleware 接得到。放在最前面，後面任何地方（OpenIddict、Controller、IdentityProviderClient）丟出的例外都會被它接住。
- OnStarting：註冊一個「回應標頭即將送出時」才執行的動作。例外處理器清空標頭發生在送出之前，所以 OnStarting 會在清空之後才執行，500 回應也能帶到安全標頭。
- context.TraceIdentifier：ASP.NET Core 為每個請求自動產生的唯一編號。同一次請求在回應和 log 裡看到的是同一個值。
- error = "server_error"：這是 OAuth 標準裡「伺服器內部錯誤」的錯誤代碼。MCP Client 看到它，就知道是 Server 端的問題，不是自己的請求有錯。
- log 會出現兩次：ASP.NET Core 的例外處理器本身也會記一筆 An unhandled exception has occurred...。我們自己再記一筆，是為了確保 TraceId 一定出現在 log 訊息裡，方便搜尋。
- 我們自己回的 400（例如「無效或已過期的 nonce」）不受影響：那些是已經處理過的錯誤，不是例外。這一步只處理沒有被 catch 的例外。

測試時要注意：這個處理器只在非開發環境啟用，所以要暫時讓它在開發環境也生效，再故意製造一個例外。測完兩處都要改回來。

1. 暫時把 if (!app.Environment.IsDevelopment()) 改成 if (true)，重啟 Authorization Server。
2. 暫時把 Redeem 網址改成 HTTP，讓 IdentityProviderClient 因為「不是 HTTPS」而丟出例外：
UPDATE "IdentityProviders" SET "RedeemApiUrl" = 'http://localhost:16888/api/internal/sso/redeem' WHERE "ProviderId" = 'pnp';
3. 開著 Follow redirects，送一次 /connect/authorize。最後會停在 bridge-callback，回 500。
4. 改回來：
UPDATE "IdentityProviders" SET "RedeemApiUrl" = 'https://localhost:16888/api/internal/sso/redeem' WHERE "ProviderId" = 'pnp';
   if (true) 也改回 if (!app.Environment.IsDevelopment())，重啟。

請執行後確認：
1. dotnet build 正常編譯。
2. 第 3 步的回應是 500，body 只有 {"error":"server_error","error_description":"伺服器發生錯誤，請稍後再試。","trace_id":"..."}，沒有例外訊息或 stack trace。
{% asset_image https_error1.jpg https_error1 %}
3. 500 回應的標頭裡仍然有 Content-Security-Policy、Referrer-Policy、X-Content-Type-Options。
{% asset_image https_error2.jpg https_error2 %}
4. Authorization Server 的 Console 有 Unhandled exception. TraceId=...，TraceId 和回應的 trace_id 一樣，下面還有完整的例外訊息「身分來源 'pnp' 的 API 網址未設定或不是 HTTPS。」。
{% asset_image https_error3.jpg https_error3 %}
5. 改回來之後，正常流程恢復。