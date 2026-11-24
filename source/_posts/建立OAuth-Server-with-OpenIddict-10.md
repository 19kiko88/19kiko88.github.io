---
title: 建立OAuth Server with OpenIddict(10)
date: 2026-11-24 13:57:55
password: 7533967
categories:
  - 後端開發
tags:
  - OAuth
  - OpenIddict
---
##### 建立 PCBRepairNavi.MCP
<!-- more -->
這一步在做什麼

建立 .NET 10 的 MCP Server 專案，先放一個不呼叫 PNP、不驗 JWT 的 ping 工具，用 curl 確認 MCP 協定能通。後面的 JWT 驗證、呼叫 PNP 都會疊在這個骨架上。

我查過了，ModelContextProtocol.AspNetCore 目前最新是 2.4.0（2026-11-07 發布），和先前殘留的 1.2.0 差了一個主版本。2.x 有一個重要變化：HTTP 傳輸預設是 Stateless，因為 MCP 規格 2026-07-28 版已經移除 session。
#
#
#
---

1. 建立專案並安裝套件
``` bash
dotnet new web -n PCBRepairNavi.MCP -o PCBRepairNavi.MCP -f net10.0
dotnet add .\PCBRepairNavi.MCP package ModelContextProtocol.AspNetCore --version 2.4.0
```

## 固定 port：PCBRepairNavi.MCP/Properties/launchSettings.json
把 https profile 的 applicationUrl 改成：
``` json
"applicationUrl": "https://localhost:7290;http://localhost:5290",
```

2. 修改 PCBRepairNavi.MCP/Program.cs（整個檔案換掉）
``` csharp
using PCBRepairNavi.MCP.Tools;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddMcpServer()
    .WithHttpTransport()          // 2.4.0 預設 HttpServerSessionMode.Stateless
    .WithTools<PingTool>();

var app = builder.Build();

app.MapMcp("/mcp");

app.Run();
```

3. 新增 PCBRepairNavi.MCP/Tools/PingTool.cs
``` csharp
using System.ComponentModel;
using System.ComponentModel;
using ModelContextProtocol.Server;

namespace PCBRepairNavi.MCP.Tools;

/// <summary>
/// 最小的 MCP Tool：不呼叫 PNP、不需要權限，只用來確認 MCP Server 可以連線（第二階段第 3 步）
/// </summary>
[McpServerToolType]
public sealed class PingTool
{
    [McpServerTool, Description("Returns a fixed string to verify that the PCBRepairNavi MCP server is reachable.")]
    public static string Ping() => "pong from PCBRepairNavi.MCP";
}
```

4. 測試用的 JSON-RPC 檔案（放在 D:\Homer\Test\mcp\，repo 外面）
在powershell執行下面指令，建立測試json檔案。
``` bash
New-Item -ItemType Directory -Force D:\Homer\Test\mcp | Out-Null
$utf8 = New-Object System.Text.UTF8Encoding($false)
[IO.File]::WriteAllText('D:\Homer\Test\mcp\init.json', '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"curl","version":"1.0"}}}', $utf8)
[IO.File]::WriteAllText('D:\Homer\Test\mcp\list.json', '{"jsonrpc":"2.0","id":2,"method":"tools/list"}', $utf8)
```

幾個重點說明

1. 先不加進 PCBRepairNavi.sln（和交接文件不同）：交接文件寫「加入 PCBRepairNavi.sln」，但「PNP 的 CI／建置機是否有 .NET 10 SDK」還沒確認。如果建置機用 msbuild PCBRepairNavi.sln 建置整個方案、又沒有 .NET 10，PNP 的建置會失敗，你說過不要影響現有功能。所以先用 dotnet run 單獨執行，等確認建置機的狀況、要發 PR 之前再決定。
2. MapMcp("/mcp")：MCP 端點固定在 /mcp。交接文件第 4.2 節第 5 項提到，端點的完整網址 = MCP Client 送的 resource = Protected Resource Metadata 的 resource = token 的 aud，這四個值必須一致。所以這一步就先固定成 https://localhost:7290/mcp，之後不要再改。
3. port 7290：避開 Authorization Server（7280）和 MockPnp（16888）。用固定的 port，是因為它會變成第 2 點那個網址的一部分。
4. Stateless：每個請求各自獨立，伺服器不記住「哪個 client 連過」，就像 PNP 的內部 API 一樣。好處是之後可以多台部署，不需要 session 黏著。我們的工具都是「查一次、回一次」，不需要 session。
5. [McpServerToolType]、[McpServerTool]、Description：MCP Client（例如 Claude）是靠 Description 決定什麼時候要呼叫這個工具，所以要寫清楚工具做什麼。這和 MVC 的 [Route] 有點像，都是「宣告」，由框架掃描後註冊。
6. NuGet 一律指定版本（交接文件第 0.1 節）：--version 2.4.0。


測試步驟

開兩個 PowerShell 視窗。

視窗 1：啟動
``` bash
cd D:\Homer\Project\PCBNavigatePortal\PCBRepairNavi.MCP
dotnet run --launch-profile https
```
視窗 2：呼叫

| # | 指令 | 驗證什麼 |
|:---:|---|---|
| 1 | `curl.exe -s -i -X POST https://localhost:7290/mcp -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' --data-binary '@D:\Homer\Test\mcp\init.json'` | MCP 端點在 `/mcp`，initialize 握手成功 |
| 2 | `curl.exe -s -i -X POST https://localhost:7290/mcp -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' -H 'MCP-Protocol-Version: 2025-06-18' --data-binary '@D:\Homer\Test\mcp\list.json'` | 工具有被註冊，並且確認 SDK 給這個工具取的名稱（下一步呼叫時要用） |


請執行後確認

- [ ] dotnet add package 安裝的是 2.4.0（沒有出現版本錯誤）
- [ ] dotnet run 顯示 Now listening on: https://localhost:7290
- [ ] #1：HTTP/1.1 200，回應裡有 "result"，裡面有 serverInfo、protocolVersion、capabilities。回應可能是 JSON，也可能是 event: message 開頭的 SSE 格式，兩種都正常
- [ ] #2：HTTP/1.1 200，"tools" 裡有一個工具，請告訴我它的 name 是 ping 還是 Ping
- [ ] 如果 #1 出現 TLS 憑證錯誤，或 #2 回錯誤，請貼上完整輸出

**ping 工具正常、名稱是 ping。MCP Server 的骨架就完成了。**
#
#
#
---
## JWT 驗證、401 與 Protected Resource Metadata
這一步在做什麼

讓 /mcp 只接受 Authorization Server 簽發的有效 access_token。沒帶 token 或 token 無效時回 401，並告訴 MCP Client「去哪裡登入」。ping 工具改成回傳 token 裡的使用者（sub），證明使用者身分有傳進工具。

這一步只用 curl 測試，token 用你之前在 Insomnia 手動跑 /connect/authorize → /connect/token 的方式取得（交接文件第 5 節）。接 MCP Inspector、驗證第 4.2 節的相容性風險，放在下一步。

1. 安裝套件
``` bash
dotnet add .\PCBRepairNavi.MCP package Microsoft.AspNetCore.Authentication.JwtBearer --version 10.0.15
```

2. 修改 PCBRepairNavi.MCP/appsettings.Development.json（整個檔案換掉）
``` json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.AspNetCore.Authentication": "Information"
    }
  },
  "Auth": {
    "Authority": "https://localhost:7280/",
    "Audience": "urn:pnp:ucs-mcp",
    "ScopesSupported": [ "ucs.ecn-rcn-info.read" ]
  }
}
```

3. 修改 PCBRepairNavi.MCP/Program.cs（整個檔案換掉）
``` csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;
using ModelContextProtocol.AspNetCore.Authentication;
using PCBRepairNavi.MCP.Tools;

var builder = WebApplication.CreateBuilder(args);

var authority = builder.Configuration["Auth:Authority"]
    ?? throw new InvalidOperationException("找不到設定 'Auth:Authority'");
var audience = builder.Configuration["Auth:Audience"]
    ?? throw new InvalidOperationException("找不到設定 'Auth:Audience'");
var scopesSupported = builder.Configuration.GetSection("Auth:ScopesSupported").Get<string[]>() ?? [];

builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;   // 驗證：JWT
    options.DefaultChallengeScheme = McpAuthenticationDefaults.AuthenticationScheme; // 401：加上 resource_metadata
})
.AddJwtBearer(options =>
{
    // 啟動後從 {Authority}.well-known/openid-configuration 取得 JWKS 網址，再依 JWT header 的 kid 找公鑰
    options.Authority = authority;
    options.MapInboundClaims = false;   // 保留 sub、scope 原本的名稱，不轉成 .NET 的長 URI
    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuer = true,
        ValidIssuer = authority,
        ValidateAudience = true,
        ValidAudience = audience,
        ValidateLifetime = true,
        ValidateIssuerSigningKey = true,
        ClockSkew = TimeSpan.FromSeconds(30),
        NameClaimType = "sub",
    };
})
.AddMcp(options =>
{
    // Protected Resource Metadata（RFC 9728）：告訴 MCP Client 要去哪個 Authorization Server 登入
    options.ResourceMetadata = new()
    {
        AuthorizationServers = { authority },
        ScopesSupported = [.. scopesSupported],
    };
});

builder.Services.AddAuthorization();
builder.Services.AddHttpContextAccessor();

builder.Services.AddMcpServer()
    .WithHttpTransport()          // 2.4.0 預設 HttpServerSessionMode.Stateless
    .WithTools<PingTool>();

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

app.MapMcp("/mcp").RequireAuthorization();

app.Run();
```

4. 修改 PCBRepairNavi.MCP/Tools/PingTool.cs（整個檔案換掉）
``` csharp
using System.ComponentModel;
using ModelContextProtocol.Server;

namespace PCBRepairNavi.MCP.Tools;

/// <summary>
/// 最小的 MCP Tool：不呼叫 PNP。回傳 access_token 的 sub，用來確認 JWT 驗證後的使用者有傳進工具
/// </summary>
[McpServerToolType]
public sealed class PingTool
{
    [McpServerTool, Description("Returns a fixed string and the signed-in user to verify that the PCBRepairNavi MCP server is reachable.")]
    public static string Ping(IHttpContextAccessor httpContextAccessor)
    {
        var sub = httpContextAccessor.HttpContext?.User.FindFirst("sub")?.Value ?? "(unknown)";
        return $"pong from PCBRepairNavi.MCP, user={sub}";
    }
}
```

5. 新增測試檔 call.json
``` bash
$utf8 = New-Object System.Text.UTF8Encoding($false)
[IO.File]::WriteAllText('D:\Homer\Test\mcp\call.json', '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"ping","arguments":{}}}', $utf8)
```

幾個重點說明

1. 兩個驗證方案分工：
   - JwtBearer 負責驗 token：用 JWKS 公鑰驗簽，再檢查 iss、aud、exp（交接文件第 4 節）。
   - AddMcp() 負責驗證失敗時怎麼回應：401 加上 WWW-Authenticate: Bearer resource_metadata="..."，並發布 metadata。
   - DefaultAuthenticateScheme 和 DefaultChallengeScheme 就是在指定這個分工。
2. Resource Server 只需要 Authorization Server 的網址，不需要任何密鑰：JWT 是用 Authorization Server 的私鑰簽的，MCP Server 從公開的 JWKS 拿公鑰來驗。這就是前面說的「MCP Server ↔ OAuth Server 之間不是 HMAC」。詳細原理見 OAuth repo 的 docs/JWT-JWKS-kid.md。
3. Audience 暫時用 urn:pnp:ucs-mcp：這是 Authorization Server 目前發出的 aud，先讓 curl 測試能通。交接文件第 4.2 節第 3 項提到，接上真正的 MCP Client 後，很可能要改成 https://localhost:7290/mcp，兩邊同步修改。所以放在設定檔，不寫死在程式裡。
4. MapInboundClaims = false：.NET 預設會把 sub 改名成 http://schemas.xmlsoap.org/.../nameidentifier。關掉之後，FindFirst("sub") 和下一步要讀的 scope 都維持原名。
5. RequireAuthorization() 只套在 /mcp：metadata 端點（/.well-known/...）由 AddMcp() 處理，不需要 token。MCP Client 在登入前就要能讀到它。
6. ScopesSupported 只放這個 Server 會用到的 scope：metadata 是公開的。目前只做 ECN/RCN，所以只列一個，之後加工具時再加。
7. ClockSkew = 30 秒：JWT 的 exp 預設容許 5 分鐘時鐘差，太寬鬆。Authorization Server 和 MCP Server 會跑在有對時的主機上，30 秒就夠了。
8. IHttpContextAccessor 當成工具的參數：SDK 會從 DI 自動帶入。工具因此可以讀到「這次呼叫的使用者是誰」，下一步呼叫 PNP 時要送的 userName 就是從這裡取得。

測試步驟

先啟動 PostgreSQL（開機自動啟動）、Authorization Server、MockPnp（交接文件第 5 節），再啟動 MCP Server：
``` bash
cd D:\Homer\Project\PCBNavigatePortal\PCBRepairNavi.MCP
dotnet run --launch-profile https
```
用 Insomnia 跑一次手動流程取得 access_token，scope 要包含 ucs.ecn-rcn-info.read。然後在另一個 PowerShell 視窗：
``` bash
$token = '<access_token>'
cd D:\Homer\Test\mcp
$h = @('-H', 'Content-Type: application/json', '-H', 'Accept: application/json, text/event-stream', '-H', 'MCP-Protocol-Version: 2025-06-18')
```
{% asset_image test_bash.jpg test_bash%}


| # | 指令 | 驗證什麼 |
|:---:|---|---|
| 1 | `curl.exe -s -i -X POST https://localhost:7290/mcp @h --data-binary '@init.json'` | 沒帶 token 會回 401，WWW-Authenticate 指向 metadata |
| 2 | `curl.exe -s https://localhost:7290/.well-known/oauth-protected-resource/mcp` | metadata 不需要 token 就能讀，內容正確 |
| 3 | `curl.exe -s -i -X POST https://localhost:7290/mcp @h -H "Authorization: Bearer $token" --data-binary '@call.json'` | 有效的 token 可以呼叫工具，sub 有傳進工具 |
| 4 | 把 `$token` 的最後一個字元改掉，再執行 #3 | 簽章被竄改的 token 會被拒絕 |

- 測試1.
參數沒有附上acess_token，回應401
{% asset_image mcp_test_1.jpg mcp_test_1%}

- 測試2.
查詢Discovery metadata
{% asset_image mcp_test_2.jpg mcp_test_2%}

- 測試3.
成功呼叫MCP Tool
{% asset_image mcp_test_3.jpg mcp_test_3%}

- 測試4.
測試用錯誤的acess_token呼叫MCP Server
{% asset_image mcp_test_4.jpg mcp_test_4%}


請執行後確認

- [ ] dotnet add package 安裝的是 10.0.15
- [ ] #1：HTTP/1.1 401，有一行 WWW-Authenticate: Bearer resource_metadata="https://localhost:7290/.well-known/oauth-protected-resource/mcp"
- [ ] #2：JSON 裡 resource 是 https://localhost:7290/mcp，authorization_servers 是 ["https://localhost:7280/"]，scopes_supported 有 ucs.ecn-rcn-info.read
- [ ] #3：HTTP/1.1 200，結果裡有 pong from PCBRepairNavi.MCP, user=test\\mockuser
- [ ] #4：HTTP/1.1 401，MCP Server 的主控台有 Bearer was not authenticated 之類的訊息，並寫出原因（例如 signature validation failed）
- [ ] 如果 #3 是 401，請貼上 MCP Server 主控台的那一行錯誤訊息（不要貼 token）