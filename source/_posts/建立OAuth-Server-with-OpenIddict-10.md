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
##### 把 PNP 的能力包成需要 OAuth 登入的 MCP Server
<!-- more -->
這一步在做什麼

建立 .NET 10 的 MCP Server 專案，先放一個不呼叫 PNP、不驗 JWT 的 ping 工具，用 curl 確認 MCP 協定能通。後面的 JWT 驗證、呼叫 PNP 都會疊在這個骨架上。

我查過了，ModelContextProtocol.AspNetCore 目前最新是 2.4.0（2026-11-07 發布），和先前殘留的 1.2.0 差了一個主版本。2.x 有一個重要變化：HTTP 傳輸預設是 Stateless，因為 MCP 規格 2026-07-28 版已經移除 session。
#
#
#
---
## 建立 PCBRepairNavi.MCP（最小可執行的 MCP Server）

1. 建立專案並安裝套件
``` bash
dotnet new web -n PCBRepairNavi.MCP -o PCBRepairNavi.MCP -f net10.0
dotnet add .\PCBRepairNavi.MCP package ModelContextProtocol.AspNetCore --version 2.4.0
```

2. 固定 port：PCBRepairNavi.MCP/Properties/launchSettings.json
把 https profile 的 applicationUrl 改成：
``` json
"applicationUrl": "https://localhost:7290;http://localhost:5290",
```

3. 修改 PCBRepairNavi.MCP/Program.cs（整個檔案換掉）
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

4. 新增 PCBRepairNavi.MCP/Tools/PingTool.cs
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

5. 測試用的 JSON-RPC 檔案（放在 D:\Homer\Test\mcp\，repo 外面）
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
MCP Server 的主控台有 Bearer was not authenticated 之類的訊息
{% asset_image error_token.png error_token%}


請執行後確認

- [ ] dotnet add package 安裝的是 10.0.15
- [ ] #1：HTTP/1.1 401，有一行 WWW-Authenticate: Bearer resource_metadata="https://localhost:7290/.well-known/oauth-protected-resource/mcp"
- [ ] #2：JSON 裡 resource 是 https://localhost:7290/mcp，authorization_servers 是 ["https://localhost:7280/"]，scopes_supported 有 ucs.ecn-rcn-info.read
- [ ] #3：HTTP/1.1 200，結果裡有 pong from PCBRepairNavi.MCP, user=test\\mockuser
- [ ] #4：HTTP/1.1 401，MCP Server 的主控台有 Bearer was not authenticated 之類的訊息，並寫出原因（例如 signature validation failed）
- [ ] 如果 #3 是 401，請貼上 MCP Server 主控台的那一行錯誤訊息（不要貼 token）
#
#
#
---
## 用 Claude Code 跑通 OAuth 登入，驗證第 4.2 節的相容性風險

這一步在做什麼

讓 Claude Code 扮演真正的 MCP Client，從「沒有 token」開始，自己完成整個流程：

1. 讀 401 的 WWW-Authenticate，找到 Protected Resource Metadata
2. 從 metadata 找到 Authorization Server
3. 開瀏覽器登入（經過 MockPnp 的 bridge）
4. 拿到 token 後呼叫 ping

這一步不改 MCP Server 的程式，只改設定，目的是觀察流程會卡在哪裡。卡住也是有用的結果，代表找到了要回 OAuth repo 處理的問題。


1. OAuth repo：註冊 Claude Code 的 redirect URI

修改 D:\Homer\Project\OAuth\OAuth.AuthorizationServer\registration.Development.json，在 pnp-mcp-client 的 redirectUris 加上一筆：
``` json
"redirectUris": [ "https://localhost:5055/callback", "http://localhost:5056/callback" ],
```

執行註冊工具，然後重新啟動 Authorization Server（附錄 B.4）：
``` bash
cd D:\Homer\Project\OAuth
dotnet run --project OAuth.Registration -- OAuth.AuthorizationServer\registration.Development.json
```

2. 匯出開發憑證給 Claude Code 信任（第 4.2 節第 4 項）
``` bash
New-Item -ItemType Directory -Force D:\Homer\Test\certs | Out-Null
dotnet dev-certs https --export-path D:\Homer\Test\certs\localhost-dev.pem --format Pem --no-password
Remove-Item D:\Homer\Test\certs\localhost-dev.key   # 私鑰用不到，匯出後立刻刪除
```

3. 在 Claude Code 加入 MCP Server

開一個新的 PowerShell 視窗，不要在目前這個 Claude Code 對話裡執行：
``` bash
$env:NODE_EXTRA_CA_CERTS = 'D:\Homer\Test\certs\localhost-dev.pem'
cd D:\Homer\Test\mcp
claude mcp add --transport http pnp-mcp https://localhost:7290/mcp --client-id pnp-mcp-client --callback-port 5056
```
{% asset_image add_mcp.jpg add_mcp%}

4. 登入並呼叫

在同一個視窗執行 claude，然後：
1. 輸入 /mcp，選 pnp-mcp，選 Authenticate。
{% asset_image pnp_mcp.jpg pnp_mcp %}

2. 瀏覽器會打開登入流程（MockPnp 會自動視為 test\mockuser 已登入），完成後回到 Claude Code。
3. 輸入：請呼叫 pnp-mcp 的 ping 工具。


幾個重點說明

1. --client-id pnp-mcp-client：直接用 Authorization Server 已經註冊的 public client，不走 DCR。這是在驗證第 4.2 節第 2 項。
2. --callback-port 5056：Claude Code 會在本機 5056 port 接收登入完成後的回呼，所以 redirect URI 是 http://localhost:5056/callback。註冊工具允許 http://localhost 這種 loopback 位址（附錄 B.2），因為回呼只在你自己的電腦上發生。
3. NODE_EXTRA_CA_CERTS：.NET 的開發憑證只被加進 Windows 的憑證存放區，Claude Code 預設不信任它，會連不上 https://localhost:7290、7280。只匯出公開憑證就夠了，私鑰要刪掉。
4. 在 D:\Homer\Test\mcp 執行 claude mcp add：預設的 local 範圍是「這個資料夾專用」，所以不會影響 PNP 專案或其他資料夾的 Claude Code 設定。之後要移除時，在同一個資料夾執行 claude mcp remove pnp-mcp。
5. 這一步預期可能會卡住。下表是可能卡住的地方，以及要看哪裡：

| 可能卡住的地方 | 對應的風險 | 現象 | 看哪裡 |
|---|---|---|---|
| ① 找不到 Authorization Server 的 metadata | 第 4.1 節：Authorization Server 只發布 `openid-configuration`，沒有 `oauth-authorization-server` | /mcp 顯示錯誤，還沒開瀏覽器 | Claude Code 的錯誤訊息 |
| ② Authorization Server 拒絕 resource 參數 | 第 4.2 節第 3 項 | 瀏覽器停在 Authorization Server 的錯誤頁（例如 `invalid_target`） | 瀏覽器網址列的 `error=`、Authorization Server 主控台 |
| ③ redirect URI 不符 | — | 瀏覽器顯示 `invalid_request` / `redirect_uri` 錯誤 | Authorization Server 主控台 |
| ④ token 的 aud 不被接受 | 第 4.2 節第 3 項 | 登入成功，但呼叫 ping 時又要求驗證 | MCP Server 主控台的 `Bearer was not authenticated` 原因 |
| ⑤ 憑證不被信任 | 第 4.2 節第 4 項 | 錯誤訊息有 `certificate`、`self-signed` | Claude Code 的錯誤訊息 |

測試步驟

前置：PostgreSQL、Authorization Server（重新啟動過）、MockPnp、MCP Server 都在執行中。

1. 執行註冊工具：驗證 redirect URI 寫入成功（回傳碼 0）。
2. 在 Claude Code 跑 /mcp → Authenticate → 登入：這是在驗證 metadata 探索、resource 參數、redirect URI（風險 ①②③⑤）。
  - 出現了② Authorization Server 拒絕 resource 參數，我們會在下個流程處理。
  {% asset_image invalid_target.jpg invalid_target%}
3. 呼叫 ping：這是在驗證 token 的 aud 被 MCP Server 接受，sub 也有傳進工具（風險 ④）。

請執行後確認

- [ ] 註冊工具執行成功
- [ ] /mcp 裡 pnp-mcp 的狀態從「需要驗證」變成「已連線」
- [ ] 呼叫 ping 得到 pong from PCBRepairNavi.MCP, user=test\mockuser
- [ ] 如果卡住：告訴我卡在上表的哪一個位置，並貼上
  - Claude Code 顯示的錯誤訊息
  - 瀏覽器最後停留頁面的網址（把 code=、state= 的值遮掉）
  - Authorization Server 和 MCP Server 主控台在那個時間點的訊息（不要貼 token）
#
#
#
---
## OAuth repo：讓 Authorization Server 接受 resource 參數

這一步在做什麼

修改 Authorization Server：

1. 登記 MCP Server 的網址為合法的 resource。
2. 讓 client 擁有要求這個 resource 的權限。
3. 讓 token 的 aud 變成 MCP Server 的網址。

MCP Server 這邊只改一行設定。照交接文件的原則，「需要調整 Authorization Server 時，先回 OAuth repo 處理」。

1. OAuth.AuthorizationServer/Program.cs
在 .AddServer(options => { ... }) 裡面，options.SetAccessTokenLifetime(...) 的前面加入：
``` csharp
// RFC 8707：MCP Client 登入時會帶 resource=<MCP Server 網址>。
// OpenIddict 只接受事先登記的 resource，未登記的回 invalid_target（ID2190）
options.RegisterResources(builder.Configuration.GetSection("OAuth:Resources").Get<string[]>() ?? []);
```

2. OAuth.AuthorizationServer/appsettings.Development.json
在最外層的 { } 裡加入一個區塊（和其他區塊並列，注意逗號）：
``` json
  "OAuth": {
    "Resources": [ "https://localhost:7290/mcp" ]
  }
```

3. OAuth.Core/Registration/RegistrationService.cs
在 client 迴圈裡，緊接在 foreach (var scope in c.Scopes) { descriptor.Permissions.Add(Permissions.Prefixes.Scope + scope); } 之後加入：
``` csharp
            // client 可要求的 resource = 它被授予的 scope 所屬的 Resource Server（rsrc: 權限，RFC 8707）
            var clientResources = manifest.Scopes
                .Where(s => c.Scopes.Contains(s.Name))
                .SelectMany(s => s.Resources)
                .Distinct(StringComparer.Ordinal);
            foreach (var resource in clientResources)
            {
                descriptor.Permissions.Add(Permissions.Prefixes.Resource + resource);
            }
```

4. OAuth.AuthorizationServer/registration.Development.json
三個 scope 的 resources 都改成 MCP Server 的網址：
``` json
  "scopes": [
    { "name": "ucs.sn-info.read", "displayName": "SN 資料讀取", "resources": [ "https://localhost:7290/mcp" ] },
    { "name": "ucs.ecn-rcn-info.read", "displayName": "UCS ECN/RCN 資料讀取", "resources": [ "https://localhost:7290/mcp" ] },
    { "name": "pnp.L1-L2-previous-rma-info.read", "displayName": "SN自動開啟點圖 讀取", "resources": [ "https://localhost:7290/mcp" ] }
  ],
```
registration.Test.json 不改（原因見重點說明第 4 點）。

5. OAuth的部分加上resources後，MCP Server的部分也需要做修改。 PCBRepairNavi.MCP/appsettings.Development.json
``` json
"Audience": "https://localhost:7290/mcp",
```
幾個重點說明

1. RegisterResources 從設定檔讀取：MCP Server 的網址每個環境都不同（開發是 localhost:7290，正式是另一個主機），所以不寫死在程式裡。正式環境再到 appsettings.Production.json 或環境變數設定 OAuth__Resources__0。
2. rsrc: 權限從 scope 推導：client 能要求哪些 resource，由「它被授予的 scope 屬於哪些 Resource Server」決定，不用在註冊檔另外寫一份，也不會出現 scope 和 resource 對不上的情況。另一個做法是呼叫 IgnoreResourcePermissions() 直接略過這道檢查，但這樣任何 client 都能要求任何已登記的 resource，我不建議。
3. aud 不用改程式：AuthorizationController.cs:308 原本就是「依 scope 查出 Resource Server，設成 aud」。scope 的 resources 換成網址後，aud 自然就變成網址。
4. registration.Test.json 和自動化測試不動：OAuth repo 的自動化測試（AuthorizationFlowTests.cs:34）用的是 registration.Test.json，斷言 aud 是 urn:pnp:ucs-mcp，而且測試不帶 resource 參數。測試環境沒有 OAuth:Resources 設定，RegisterResources 會收到空陣列，對不帶 resource 的請求沒有影響。所以測試應該全部照常通過，這也正好用來驗證這次修改沒有破壞原本的流程。
5. 註冊工具會整筆覆蓋 client（附錄 B.3）：執行後 pnp-mcp-client 會自動帶上新的 rsrc:https://localhost:7290/mcp 權限，不需要先刪除。
6. 文件：設計文件和交接文件裡寫的 aud（urn:pnp:ucs-mcp）之後要更新，等你要求時我再整理。

測試步驟

1. OAuth repo：dotnet test。驗證原本的流程沒有被破壞（重點說明第 4 點）。
2. OAuth repo：執行註冊工具，然後重新啟動 Authorization Server。
cd D:\Homer\Project\OAuth
dotnet run --project OAuth.Registration -- OAuth.AuthorizationServer\registration.Development.json
3. PNP repo：重新啟動 MCP Server（讓新的 Audience 生效）。
4. Claude Code（D:\Homer\Test\mcp，記得先設定 NODE_EXTRA_CA_CERTS）：/mcp → pnp-mcp → 重新 Authenticate，再呼叫 ping。這一步驗證 resource 參數被接受，而且 token 的 aud 被 MCP Server 接受。

Claude Code成功呼叫到MCP Tool了
{% asset_image call_mcp_tool.jpg call_mcp_tool %}

<span style="color: red;">這是一個重要的里程碑：真正的 MCP Client（Claude Code）已經能自己完成整個 OAuth 登入並呼叫 MCP Tool。</span>

#
#
#
---
## MCP Server 用 HMAC 呼叫 PNP 的 client

這一步在做什麼

把第一階段 PowerShell 腳本做的事（簽章、送出），改寫成 MCP Server 裡的 C# 程式，讓下一步的 MCP Tool 可以直接呼叫 PNP。分成三塊：

- PnpHmacSigner：純函式，計算簽章（和 PNP 的 InternalApiSignature.ComputeSignature 是同一套演算法）。
- PnpHmacHandler：DelegatingHandler，在每個送往 PNP 的請求上自動加 X-Timestamp、X-Signature。
- PnpInternalApiClient：呼叫 api/internal/ucs/ecn-rcn，把 HTTP 狀態碼轉成結果。

1. 設定 PNP 網址：PCBRepairNavi.MCP/appsettings.Development.json
在 "Auth": { ... } 後面加入（注意逗號）：
``` json
  "Pnp": {
    "BaseUrl": "https://tp-caevm-vt01.corpnet.asus/PNP/",
    "TimeoutSeconds": 10
  }
```

2. 設定密鑰 A：user-secrets（不進 git）(先前存在InternalApi.secrets.config的那組Secret Key)
``` bash
cd D:\Homer\Project\PCBNavigatePortal\PCBRepairNavi.MCP
dotnet user-secrets init
dotnet user-secrets set "Pnp:HmacSecret" "<第一階段 A-2 產生的密鑰>"
```

3. 新增 PCBRepairNavi.MCP/Pnp/PnpOptions.cs
``` csharp
namespace PCBRepairNavi.MCP.Pnp;

/// <summary>
/// 呼叫 PNP 內部 API 的設定。BaseUrl 放 appsettings；HmacSecret（密鑰 A）放 user-secrets／正式環境的金鑰管理
/// </summary>
public sealed class PnpOptions
{
    public const string SectionName = "Pnp";

    /// <summary>含虛擬目錄、以 / 結尾，例如 https://host/PNP/</summary>
    public string BaseUrl { get; set; } = "";

    public string HmacSecret { get; set; } = "";

    public int TimeoutSeconds { get; set; } = 10;
}
```

4. 新增 PCBRepairNavi.MCP/Pnp/PnpHmacSigner.cs
``` csharp
using System.Globalization;
using System.Security.Cryptography;
using System.Text;

namespace PCBRepairNavi.MCP.Pnp;

/// <summary>
/// PNP 內部 API 的 HMAC-SHA256 簽章（OAuth repo docs/OAuth2.1-MCP-AuthServer-Design.md 第4.4節）。
/// 必須與 PNP 的 PCBRepairNavi.Helpers.InternalApiSignature.ComputeSignature 完全一致
/// </summary>
public static class PnpHmacSigner
{
    public static string ComputeSignature(string secret, long timestamp, string method, string path, string body)
    {
        var bodyHash = Convert.ToHexStringLower(SHA256.HashData(Encoding.UTF8.GetBytes(body)));
        var canonical = $"{timestamp.ToString(CultureInfo.InvariantCulture)}\n{method}\n{path}\n{bodyHash}";
        var signature = HMACSHA256.HashData(Encoding.UTF8.GetBytes(secret), Encoding.UTF8.GetBytes(canonical));
        return Convert.ToBase64String(signature);
    }
}
```

5. 新增 PCBRepairNavi.MCP/Pnp/PnpHmacHandler.cs
``` csharp
using System.Globalization;
using Microsoft.Extensions.Options;

namespace PCBRepairNavi.MCP.Pnp;

/// <summary>
/// 在每個送往 PNP 內部 API 的請求加上 X-Timestamp、X-Signature。
/// 路徑取 RequestUri.AbsolutePath（含虛擬目錄），與 PNP 端的 Request.Url.AbsolutePath 一致
/// </summary>
public sealed class PnpHmacHandler(IOptions<PnpOptions> options, TimeProvider timeProvider) : DelegatingHandler
{
    protected override async Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
    {
        var body = request.Content is null ? "" : await request.Content.ReadAsStringAsync(cancellationToken);
        var timestamp = timeProvider.GetUtcNow().ToUnixTimeSeconds();
        var path = request.RequestUri!.AbsolutePath;

        var signature = PnpHmacSigner.ComputeSignature(options.Value.HmacSecret, timestamp, request.Method.Method, path, body);

        request.Headers.Remove("X-Timestamp");
        request.Headers.Remove("X-Signature");
        request.Headers.Add("X-Timestamp", timestamp.ToString(CultureInfo.InvariantCulture));
        request.Headers.Add("X-Signature", signature);

        return await base.SendAsync(request, cancellationToken);
    }
}
```

6. 新增 PCBRepairNavi.MCP/Pnp/PnpInternalApiClient.cs
``` csharp
using System.Net;
using System.Net.Http.Json;
using System.Text;
using System.Text.Json;

namespace PCBRepairNavi.MCP.Pnp;

public enum PnpCallStatus
{
    Success,
    BadRequest,      // PNP 400：輸入格式錯誤
    Unauthorized,    // PNP 401：簽章錯誤、時鐘差、密鑰未設定 → 屬設定問題
    UpstreamFailed,  // PNP 502：UCS 失敗
    Unavailable,     // 連線失敗、逾時、其他非預期回應
}

public sealed record PnpResult<T>(PnpCallStatus Status, T? Value);

/// <summary>PNP 回傳的 ECN/RCN 資料（對應 PNP 的 ResUcsEcnRcnInfoDto）</summary>
public sealed record EcnRcnItem(
    string? ArnNo,
    string? SopFileName,
    string? SopPath,
    string? SipProblem,
    string? CreatorName,
    string? ScheduleDate,
    string? NoticeViewLink);

/// <summary>
/// 呼叫 PNP 內部 API（PCBRepairNavi InternalUcsApiController）。簽章由 PnpHmacHandler 自動加上
/// </summary>
public sealed class PnpInternalApiClient(HttpClient httpClient, ILogger<PnpInternalApiClient> logger)
{
    private sealed record EcnRcnResponse(List<EcnRcnItem>? Items);

    public async Task<PnpResult<IReadOnlyList<EcnRcnItem>>> GetEcnRcnAsync(string bomNo, string userName, CancellationToken cancellationToken)
    {
        // 相對路徑不以 / 開頭，才會接在 BaseUrl 的虛擬目錄後面
        const string relativePath = "api/internal/ucs/ecn-rcn";
        var body = JsonSerializer.Serialize(new { bomNo, userName });

        try
        {
            using var content = new StringContent(body, Encoding.UTF8, "application/json");
            using var response = await httpClient.PostAsync(relativePath, content, cancellationToken);

            switch (response.StatusCode)
            {
                case HttpStatusCode.OK:
                    var payload = await response.Content.ReadFromJsonAsync<EcnRcnResponse>(cancellationToken);
                    return new(PnpCallStatus.Success, payload?.Items ?? []);
                case HttpStatusCode.BadRequest:
                    logger.LogWarning("PNP ecn-rcn rejected input. BomNo={BomNo}", bomNo);
                    return new(PnpCallStatus.BadRequest, null);
                case HttpStatusCode.Unauthorized:
                    logger.LogError("PNP ecn-rcn returned 401. Check Pnp:HmacSecret, Pnp:BaseUrl (virtual directory) and clock skew (±5s).");
                    return new(PnpCallStatus.Unauthorized, null);
                case HttpStatusCode.BadGateway:
                    logger.LogWarning("PNP ecn-rcn upstream (UCS) failed. BomNo={BomNo}", bomNo);
                    return new(PnpCallStatus.UpstreamFailed, null);
                default:
                    logger.LogError("PNP ecn-rcn unexpected status {StatusCode}. Location={Location}",
                        (int)response.StatusCode, response.Headers.Location);
                    return new(PnpCallStatus.Unavailable, null);
            }
        }
        catch (Exception ex) when (ex is HttpRequestException or TaskCanceledException or JsonException
                                   && !cancellationToken.IsCancellationRequested)
        {
            logger.LogError(ex, "PNP ecn-rcn call failed.");
            return new(PnpCallStatus.Unavailable, null);
        }
    }
}
```

7. 修改 PCBRepairNavi.MCP/Program.cs
開頭加上：
``` csharp
using Microsoft.Extensions.Options;
using PCBRepairNavi.MCP.Pnp;
```

在 builder.Services.AddMcpServer() 之前加入：
``` csharp
// ---------- 呼叫 PNP 內部 API（HMAC 密鑰 A） ----------
builder.Services.AddOptions<PnpOptions>()
    .Bind(builder.Configuration.GetSection(PnpOptions.SectionName))
    .Validate(o => Uri.TryCreate(o.BaseUrl, UriKind.Absolute, out var uri)
                   && uri.Scheme == Uri.UriSchemeHttps
                   && o.BaseUrl.EndsWith('/'),
              "Pnp:BaseUrl 必須是 https:// 開頭、以 / 結尾的絕對網址（含虛擬目錄）")
    .Validate(o => !string.IsNullOrEmpty(o.HmacSecret), "找不到設定 'Pnp:HmacSecret'（請以 user-secrets 設定）")
    .ValidateOnStart();

builder.Services.AddSingleton(TimeProvider.System);
builder.Services.AddTransient<PnpHmacHandler>();
builder.Services.AddHttpClient<PnpInternalApiClient>((sp, client) =>
    {
        var options = sp.GetRequiredService<IOptions<PnpOptions>>().Value;
        client.BaseAddress = new Uri(options.BaseUrl);
        client.Timeout = TimeSpan.FromSeconds(options.TimeoutSeconds);
    })
    // 不自動跟隨轉址：302 代表網址設錯（http→https、被導向登入頁），跟隨後 POST 會變 GET、簽章也不再有效
    .ConfigurePrimaryHttpMessageHandler(() => new SocketsHttpHandler { AllowAutoRedirect = false })
    .AddHttpMessageHandler<PnpHmacHandler>();
```

8. 新增測試專案 PCBRepairNavi.MCP.Tests
``` bash
cd D:\Homer\Project\PCBNavigatePortal
dotnet new xunit -n PCBRepairNavi.MCP.Tests -o PCBRepairNavi.MCP.Tests -f net10.0
dotnet add .\PCBRepairNavi.MCP.Tests reference .\PCBRepairNavi.MCP\PCBRepairNavi.MCP.csproj
Remove-Item .\PCBRepairNavi.MCP.Tests\UnitTest1.cs
```

新增 PCBRepairNavi.MCP.Tests/PnpHmacSignerTests.cs：
``` csharp
using PCBRepairNavi.MCP.Pnp;

namespace PCBRepairNavi.MCP.Tests;

public class PnpHmacSignerTests
{
    // 與 PNP 的 PCBRepairNavi.Tests/InternalApiSignatureTests 使用同一組對照值：
    // 兩邊都算出同一個簽章，代表 MCP Server（簽章端）與 PNP（驗章端）格式一致
    private const string Secret = "unit-test-secret";
    private const long Timestamp = 1793180718;
    private const string Path = "/PNP/api/internal/ucs/ecn-rcn";
    private const string Body = "{\"bomNo\":\"60NB0000-MB0000\",\"userName\":\"mockuser\"}";
    private const string ExpectedSignature = "mLzSTk8tcA7AW6fLcGRz2Pt7Y0ATEbqBhNHPpiYdgbs=";

    [Fact]
    public void ComputeSignature_MatchesPnpVector()
    {
        Assert.Equal(ExpectedSignature, PnpHmacSigner.ComputeSignature(Secret, Timestamp, "POST", Path, Body));
    }

    [Fact]
    public void ComputeSignature_DifferentPath_ChangesSignature()
    {
        Assert.NotEqual(ExpectedSignature, PnpHmacSigner.ComputeSignature(Secret, Timestamp, "POST", "/api/internal/ucs/ecn-rcn", Body));
    }
}
```

幾個重點說明

1. 對照值是同一個 mLzSTk8...：第一階段 PNP 端的測試證明「PNP 驗章 = .NET 10 寫法」，這次 MCP 端的測試證明「MCP 簽章 = 同一個值」。兩端各自對同一個標準答案，所以不用部署就能確定兩邊格式一致。
2. DelegatingHandler：就像在 HttpClient 前面加了一個自動蓋章的關卡。之後新增其他 ucs.* 工具時，只要寫 PostAsync(...)，不用每次自己簽章，也不會忘記。
3. 路徑取 request.RequestUri.AbsolutePath：和第一階段腳本、PNP 端的 Request.Url.AbsolutePath 是同一個概念，都取「實際送出的網址」，所以虛擬目錄、大小寫一定一致。
4. BaseUrl 以 / 結尾、相對路徑不以 / 開頭：先前說明虛擬目錄時提過的 Uri 合併規則。啟動時用 Validate 檢查，設定錯了就不讓網站啟動，比跑到一半才 401 好查得多。
5. ValidateOnStart()：缺密鑰或網址不是 https 時，dotnet run 會直接報錯並寫出原因。也順便強制做到設計文件第 4.2 節的「只接受 HTTPS」。
6. AllowAutoRedirect = false：HttpClient 預設會跟隨 302。第一階段遇過的 http→https 轉址、/api 外被導向登入頁，跟隨後 POST 會變成 GET，最後只會看到莫名其妙的錯誤。關掉之後，302 會直接落到 default 分支並記錄 Location，一眼就知道被導去哪裡。
7. PnpCallStatus：把 PNP 的狀態碼轉成有意義的結果，讓下一步的 MCP Tool 可以回覆不同的訊息給使用者（例如「查無此料號格式」和「系統暫時無法使用」）。401 記為 Error，因為它代表設定錯誤，不是使用者的問題。
8. TimeProvider：.NET 8 起內建的「可替換時鐘」，和第一階段 Verify(..., nowUnixSeconds) 的想法相同，之後要測時間相關的邏輯時可以替換。
9. 測試專案同樣先不加進 PCBRepairNavi.sln：和 MCP 專案的理由相同，建置機是否有 .NET 10 還沒確認。dotnet new xunit 範本產生的套件版本都是明確指定的。

測試步驟

1. dotnet test .\PCBRepairNavi.MCP.Tests：這一步驗證 MCP 端的簽章和 PNP 端一致。
2. 刻意改壞：把 PnpHmacSigner 的 ToHexStringLower 改成 ToHexString（大寫）後重跑測試，確認會失敗，再改回來。這一步驗證測試真的能抓到格式差異。
3. 啟動驗證：
   - 先執行 dotnet user-secrets remove "Pnp:HmacSecret"，再 dotnet run --launch-profile https：應該啟動失敗，並顯示「找不到設定 'Pnp:HmacSecret'」。
   - 把密鑰設回去，再啟動一次：應該正常啟動。

   這一步驗證設定錯誤時會在啟動階段就被擋下。

請執行後確認

- [ ] 2 個測試全部通過
- [ ] 改成大寫後，ComputeSignature_MatchesPnpVector 失敗；改回來後恢復通過
- [ ] 沒有密鑰時，啟動失敗並顯示 Pnp:HmacSecret 的訊息
- [ ] 設回密鑰後，正常啟動，Claude Code 的 ping 仍然可以呼叫（確認沒有影響前一步）
- [ ] git status：看不到密鑰（user-secrets 存在 %APPDATA%\Microsoft\UserSecrets\，不在專案資料夾）