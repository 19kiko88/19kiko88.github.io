---
title: 建立OAuth Server with .Net
date: 2026-10-20 14:54:47
tags:
---
##### 用.Net建立一個測試用remote MCP Server專案
<!-- more -->
#
#
#
---
## 1. 建立新的 Solution 與 Web API 專案
``` bash
mkdir D:\Homer\Project\OAuth
cd D:\Homer\Project\OAuth
dotnet new sln -n OAuth
dotnet new webapi -n OAuth.AuthorizationServer -controllers
dotnet sln add .\OAuth.AuthorizationServer\OAuth.AuthorizationServer.csproj
```
#
#
#
---

## 2. 加入 OpenIddict 相關 NuGet 套件

``` bash
dotnet add package OpenIddict.AspNetCore
dotnet add package OpenIddict.EntityFrameworkCore
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL
dotnet add package Microsoft.EntityFrameworkCore.Design

dotnet tool install --global dotnet-ef
```

套件說明（跟上次講的對應關係一樣，只是換provider）：

- OpenIddict.AspNetCore：OpenIddict 核心，掛進 ASP.NET Core pipeline，提供 Step 6 會用到的 AddOpenIddict() 設定方法。
- OpenIddict.EntityFrameworkCore：OpenIddict 的 Applications/Authorizations/Scopes/Tokens 資料模型，對應 EF Core。
- Npgsql.EntityFrameworkCore.PostgreSQL：PostgreSQL 的 EF Core Provider，提供 .UseNpgsql(connectionString) 這個方法，讓 EF Core 知道怎麼對 PostgreSQL 產生 SQL、處理 migration。
- Microsoft.EntityFrameworkCore.Design：EF Core migrations 的設計期工具，跟資料庫引擎無關，兩種DB都要裝。
- dotnet-ef：全域 CLI 工具，不分資料庫引擎都要裝一次。
#
#
#
---

## 3. 建立 DbContext 與 OpenIddict 資料模型

在專案目錄底下新增一個檔案 Data\AuthorizationServerDbContext.cs

說明：

- 繼承 DbContext 是標準 EF Core 寫法，這個類別之後會代表「跟資料庫之間的那個連線工作階段」。
- builder.UseOpenIddict() 是 OpenIddict.EntityFrameworkCore 套件提供的擴充方法，呼叫它之後，OpenIddict 內建的四張表就會自動變成這個 DbContext 的一部分，你不用自己手刻 DbSet<Application> 這些東西——OpenIddict 都幫你定義好了。
- 這個檔案目前只定義結構，還沒有實際連到PostgreSQL，連線字串跟 .UseNpgsql() 的註冊會在 Step 4（Program.cs 設定）處理。

請執行後確認：
1. 檔案建立在正確路徑，namespace 是 OAuth.AuthorizationServer.Data。
2. dotnet build 可以正常編譯（這時候 AuthorizationServerDbContext 還沒被任何地方使用，屬於正常情況，不會因為沒用到就編譯失敗）。
#
#
#
---

## 4. 設定 PostgreSQL 連線字串，把 DbContext 掛進 DI
- 在 appsettings.json 加連線字串
``` json
{
  "ConnectionStrings": {
    "AuthorizationServerDb": "Host=localhost;Port=5432;Database=oauth_authserver;Username=postgres;Password=你的密碼"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*"
}
```

- 在 Program.cs 註冊 DbContext
在 var builder = WebApplication.CreateBuilder(args); 之後、var app = builder.Build(); 之前，加入：
``` csharp
using Microsoft.EntityFrameworkCore;
using OAuth.AuthorizationServer.Data;

var connectionString = builder.Configuration.GetConnectionString("AuthorizationServerDb")
    ?? throw new InvalidOperationException("找不到連線字串 'AuthorizationServerDb'");

builder.Services.AddDbContext<AuthorizationServerDbContext>(options =>
{
    options.UseNpgsql(connectionString);
});
```
#
#
#
---

## 5：設定 OpenIddict Server 本身的行為
接續 Step 4，告訴 OpenIddict 要用剛剛那個 DbContext 存資料，並設定核心行為——只允許哪些 grant type、強制 PKCE、token 生命週期、簽章方式。這一步做完，OpenIddict 的「引擎」就設定好了，但還沒有實際處理登入橋接邏輯的程式（那是下一步要做的），所以這階段先不要急著測試打 /connect/authorize。

- 在 Program.cs，緊接著 Step 4 加的 AddDbContext 之後，加入：
``` csharp
using OpenIddict.Abstractions;

builder.Services.AddOpenIddict()
    .AddCore(options =>
    {
        // 告訴 OpenIddict 用我們的 DbContext 存 Applications/Authorizations/Scopes/Tokens
        options.UseEntityFrameworkCore()
               .UseDbContext<AuthorizationServerDbContext>();
    })
    .AddServer(options =>
    {
        // 指定 /connect/authorize、/connect/token 這兩個端點的路徑
        options.SetAuthorizationEndpointUris("connect/authorize")
               .SetTokenEndpointUris("connect/token");

        // OAuth2.1 規範：只開放 authorization_code 與 refresh_token 這兩種 grant
        // 不呼叫 AllowImplicitFlow()、不呼叫 AllowPasswordFlow()
        options.AllowAuthorizationCodeFlow()
               .AllowRefreshTokenFlow();

        // 強制所有 client 都要用 PKCE（不只 public client）
        options.RequireProofKeyForCodeExchange();

        // Token 生命週期（暫定值，先用合理預設；Refresh Token 真正的
        // 「絕對上限＝ticketExpiration」邏輯會在後面接 Bridge 時另外實作）
        options.SetAccessTokenLifetime(TimeSpan.FromMinutes(30));
        options.SetRefreshTokenLifetime(TimeSpan.FromHours(12)); // 暫時對齊PNP ticket的12小時，之後會被動態上限取代

        // 開發階段用的簽章/加密憑證：之後正式環境一定要換成
        // 真正保管好的簽章憑證（見設計文件第9節注意事項2、第8.1節）
        options.AddDevelopmentEncryptionCertificate()
               .AddDevelopmentSigningCertificate();

        // 關鍵：關掉 Access Token 加密，確保輸出是外部可驗簽的「乾淨JWT」
        // 而不是 OpenIddict 預設的加密格式（見設計文件第9節注意事項1）
        options.DisableAccessTokenEncryption();

        // 讓 /connect/authorize、/connect/token 的實際處理邏輯
        // 可以由我們自己寫的 Controller action 接手（下一步會用到）
        options.UseAspNetCore()
               .EnableAuthorizationEndpointPassthrough()
               .EnableTokenEndpointPassthrough();
    })
    .AddValidation(options =>
    {
        // 讓這個 App 自己也能驗證自己發出的 token（方便階段性測試用）
        // 之後真正獨立的 MCP Resource Server 會改用遠端 JWKS 驗證，不是這個設定
        options.UseLocalServer();
        options.UseAspNetCore();
    });
```

幾個關鍵點說明：
- AllowAuthorizationCodeFlow() + AllowRefreshTokenFlow() 是刻意只開這兩個——OAuth2.1 明確拿掉 Implicit Grant 跟 Resource Owner Password Credentials Grant，我們沒有呼叫對應的 Allow...Flow()，它們就是被禁用的狀態。
- RequireProofKeyForCodeExchange() 讓 PKCE 變成強制（不是只對 public client 選擇性開啟），符合OAuth2.1的要求。
- DisableAccessTokenEncryption() 是設計文件第9節提到的坑——沒加這行，OpenIddict 預設會把 access token 加密，外部的 MCP Resource Server 之後用 JWKS 公鑰會驗不了簽。
- AddDevelopmentSigningCertificate() 現在先用開發用的自簽憑證只是為了先讓程式跑得起來，正式上線前一定要換成正式保管的簽章憑證（對應設計文件第8.1節），這個我們會在後面的步驟特別處理，不會漏掉。
- EnableAuthorizationEndpointPassthrough() / EnableTokenEndpointPassthrough() 這兩行很重要：讓 OpenIddict 不要自己把請求整個吃掉處理完，而是解析驗證完基本協定格式後，把控制權交給我們自己的 Controller action——這是後面要接 Bridge 登入橋接邏輯的關鍵，沒開這個就沒辦法插入自訂邏輯。

請注意：這步做完後，如果你現在就手動打 /connect/authorize，會出錯或404——這是正常的，因為我們還沒寫對應的 Controller action 來接手處理（下一步才會寫）。不用懷疑這步做錯了。
#
#
#
---

## 6：建立並套用 EF Core Migration，在 PostgreSQL 建出資料表
目前為止我們只是在「程式邏輯層面」定義了資料模型（DbContext + OpenIddict的UseOpenIddict()），PostgreSQL裡實際上還沒有任何表。這一步要產生 migration 程式碼，再套用到資料庫，把 Applications、Authorizations、Scopes、Tokens 這四張表真的建出來。

``` bash
cd .\OAuth.AuthorizationServer
```

``` bash
# 產生 migration（會在專案底下新增一個 Migrations 資料夾）
dotnet ef migrations add InitialCreate
```

``` bash
# 把 migration 實際套用到你 appsettings.json 裡設定的那個 PostgreSQL
dotnet ef database update
```

說明：
- dotnet ef migrations add InitialCreate：EF Core 工具會比對目前 DbContext 的模型（因為有呼叫 UseOpenIddict()，模型裡已經包含 OpenIddict 那四張表），產生對應的 C# migration 程式碼，描述「要怎麼從空資料庫變成符合這個模型的資料庫」。這一步不會連線資料庫，純粹本機產生程式碼檔案。
- dotnet ef database update：這一步才會真的連線到你 appsettings.json 裡設定的 PostgreSQL，執行 migration、建出實際的資料表。

請執行後確認：
1. dotnet ef migrations add InitialCreate 有成功產生 Migrations 資料夾，裡面有類似 XXXXXXXX_InitialCreate.cs 的檔案。
2. dotnet ef database update 跑完沒有錯誤訊息。
3. 用任何 PostgreSQL 工具（psql、pgAdmin、DBeaver都行）連進去那個資料庫，確認看得到 OpenIddictApplications、OpenIddictAuthorizations、OpenIddictScopes、OpenIddictTokens 這幾張表。
#
#
#
---

##  7：Seed 資料 — 註冊 MCP Client 與建立 Scope
資料表結構有了，但裡面還是空的。這一步要寫一段「啟動時執行一次」的程式碼，定義好的 scope、以及PNP這個MCP client的註冊資料寫進 OpenIddictScopes、OpenIddictApplications 這兩張表。這段程式碼設計成冪等（idempotent）——每次程式啟動都會跑一次，但已經存在的資料不會重複建立，所以之後重啟服務不用擔心資料被重複塞入。

``` csharp
using OpenIddict.Abstractions;
using static OpenIddict.Abstractions.OpenIddictConstants;

await SeedDataAsync(app);

app.Run();

static async Task SeedDataAsync(WebApplication app)
{
    using var scope = app.Services.CreateScope();
    var scopeManager = scope.ServiceProvider.GetRequiredService<IOpenIddictScopeManager>();
    var appManager = scope.ServiceProvider.GetRequiredService<IOpenIddictApplicationManager>();

    var scopeDescriptors = new[]
    {
        new OpenIddictScopeDescriptor { Name = "uus.sn.read", DisplayName = "SN 資料讀取" },
        new OpenIddictScopeDescriptor { Name = "uus.ecn-rcn.read", DisplayName = "ECN/RCN 資料讀取" },
        new OpenIddictScopeDescriptor { Name = "bnb.rma-info.read", DisplayName = "RMA 資料讀取" },
    };

    foreach (var descriptor in scopeDescriptors)
    {
        if (await scopeManager.FindByNameAsync(descriptor.Name!) is null)
        {
            await scopeManager.CreateAsync(descriptor);
        }
    }

    const string mcpClientId = "bnb-mcp-client";

    if (await appManager.FindByClientIdAsync(mcpClientId) is null)
    {
        var descriptor = new OpenIddictApplicationDescriptor
        {
            ClientId = mcpClientId,
            ClientType = ClientTypes.Public,
            ConsentType = ConsentTypes.Implicit,
            DisplayName = "BNB MCP Client",
        };

        // TODO：暫時佔位網址，等實際的 MCP client 確定它的 redirect_uri 後要換成真的
        descriptor.RedirectUris.Add(new Uri("https://localhost:5055/callback"));

        descriptor.Permissions.Add(Permissions.Endpoints.Authorization);
        descriptor.Permissions.Add(Permissions.Endpoints.Token);
        descriptor.Permissions.Add(Permissions.GrantTypes.AuthorizationCode);
        descriptor.Permissions.Add(Permissions.GrantTypes.RefreshToken);
        descriptor.Permissions.Add(Permissions.ResponseTypes.Code);

        foreach (var s in scopeDescriptors)
        {
            descriptor.Permissions.Add(Permissions.Prefixes.Scope + s.Name);
        }

        await appManager.CreateAsync(descriptor);
    }
}
```

請執行後確認：
1. dotnet build 正常編譯。
2. 跑一次 dotnet run，正常啟動不報錯。
3. 查資料庫：SELECT * FROM "OpenIddictScopes"; 應該看到3筆（不是原本的5筆），SELECT * FROM "OpenIddictApplications"; 應該看到1筆bnb-mcp-client，它的Permissions欄位（JSON）裡應該包含這3個scope的permission字串。


這邊用痞客邦也可以用Google帳號登入來當範例，解釋什麼是Client ID。假設痞客邦的工程師要讓使用者使用 Google 帳號登入，<span style="color: red;">就必須先向 Google 註冊這個登入應用程式，取得一組 Client ID。</span>
例如，以下是虛構的 Client ID：
``` text
pixnet-login-123.apps.googleusercontent.com
```
這組 ID 的用途是告訴 Google：「我是痞客邦註冊的登入應用程式，現在要請使用者透過 Google 帳號進行登入。」
所以，Client ID 就像是應用程式向 Google 出示的身分識別證。當你將登入請求送往 Google時，參數就要附上痞客幫的 Client ID，來讓Goole驗證伺服器辨識是哪個應用程式。

Client ID用途如下：
1. 確認 Client 是否已註冊及可使用
根據請求中的 client_id，查詢 OpenIddict 的應用程式註冊資料，確認該 Client 是否存在，以及是否符合伺服器的啟用狀態與使用政策。
2. 驗證 Redirect URI
確認請求中的 redirect_uri(驗證身分後，要導回的頁面) 是否符合該 Client 註冊時設定的允許網址，避免授權結果被導向未經允許的網站。
3. 驗證 PKCE
如果該 Client 必須使用 PKCE，授權請求就必須包含符合規定的 code_challenge 與 code_challenge_method。之後交換授權碼時，伺服器還必須驗證 code_verifier 是否與原本的 Challenge 相符。
4. 限制 Scope 權限
確認請求中的 scope 是否在該 Client 被允許要求的範圍內，避免 Client 取得超出其註冊權限的存取範圍。
5. 限制允許的授權流程
檢查該 Client 是否被允許使用目前要求的授權流程，例如 Authorization Code Flow，以及是否符合相應的 Response Type、Endpoint 與其他安全政策。

一句話總結：Client ID 的核心用途是「識別這次請求是以哪個已註冊的應用程式名義提出」，再由 OAuth Server 根據該 Client 的註冊設定、授權流程與安全驗證機制，決定是否接受這次請求。
#
#
#
---

##   8：驗證 JWKS 與 Discovery Metadata 端點
在繼續寫更複雜的自訂登入橋接邏輯之前，先確認目前為止做的這些設定，整個OpenIddict Server本身有沒有正確跑起來。/.well-known/openid-configuration（Discovery metadata）跟 JWKS 這兩個端點是OpenIddict自動產生的，不需要我們寫任何Controller，剛好拿來當一個「前面做的東西都接對了嗎」的健檢點。

先啟動程式(用https執行，驗證伺服器只能在https上運行)
``` bash
dotnet run --launch-profile https
```

程式啟動後，看終端機印出來的網址（通常是 https://localhost:xxxx），另開一個終端機或瀏覽器測試：
（-k 是因為開發階段用的是本機自簽憑證，先跳過憑證驗證；如果你想正規一點，可以先跑 dotnet dev-certs https --trust 信任本機憑證，就不用加 -k。）
``` bash
# Discovery metadata
curl -k https://localhost:xxxx/.well-known/openid-configuration

# JWKS（公鑰集合）
curl -k https://localhost:xxxx/.well-known/jwks
```

要看到什麼才算正確：
1. /.well-known/openid-configuration 回傳的JSON裡應該要有：
   - "authorization_endpoint"：指向 .../connect/authorize
   - "token_endpoint"：指向 .../connect/token
   - "grant_types_supported"：陣列裡只有 "authorization_code" 跟 "refresh_token"，不應該出現 "implicit" 或 "password"（這是Step 6設定AllowAuthorizationCodeFlow()/AllowRefreshTokenFlow()的結果）
   - "code_challenge_methods_supported"：應該有 "S256"（PKCE強制的證明）
2. /.well-known/jwks 回傳的JSON裡應該要有一個 "keys" 陣列，裡面至少一把key，"use": "sig"（簽章用），"kty": "RSA"——這是Step 6 <span style="color: red;">AddDevelopmentSigningCertificate() 產生的開發用金鑰的公鑰部分。</span>

如果這兩個端點打不通或內容不對，代表Step 5~6的OpenIddict設定哪裡有問題，我們要先抓出來，不要帶著錯誤設定往下走到更複雜的自訂登入邏輯。

請執行後確認：
1. 兩個端點都能正常回應（不是404或500）。
2. 內容符合上面列的檢查點。

- 什麼是 .well-known/openid-configuration？
<span style="color: red;">/.well-known/openid-configuration 是標準化路徑，每個 OpenID Connect 提供者都會在此公布設定。</span>將它接在任一 OIDC issuer 的基底 URL 後面，即可取得一份 JSON，列出該提供者的端點、支援功能與密碼學能力。

例如：
Google：https://accounts.google.com/.well-known/openid-configuration
Microsoft：https://login.microsoftonline.com/{tenant}/v2.0/.well-known/openid-configuration

詳細可以參考：  [什麼是 .well-known/openid-configuration？開發者指南](https://codelabs.developers.google.com/codelabs/clasp?hl=zh-tw#0)
#
#
#
---

##   9：新增「暫存原始授權請求」的資料模型
/connect/authorize 收到請求後，我們要把使用者導去PCBRepairNavi的 /sso/bridge 驗證身分，驗證完使用者的瀏覽器會被導回我們這個Server——但中間繞去PCBRepairNavi那一圈，原本/connect/authorize帶的那些參數（client_id、redirect_uri、scope、code_challenge等）會跟著消失，瀏覽器導回來的時候我們沒辦法直接從網址上拿回它們。

所以需要一個地方暫存這次原始請求的內容，搭配一個nonce（亂數識別碼）當索引，等使用者從Bridge繞回來、帶著nonce回來時，我們才能查回「原來這次要授權的是哪個client、哪些scope」，繼續把OAuth流程走完。

在 D:\Homer\Project\OAuth\OAuth.AuthorizationServer\Data 底下新增 PendingAuthorizeRequest.cs：
``` csharp
namespace OAuth.AuthorizationServer.Data;

public class PendingAuthorizeRequest
{
    public string Nonce { get; set; } = default!;            // 主鍵，隨機產生，當作Bridge往返的識別碼
    public string OriginalRequestJson { get; set; } = default!; // 原始 /connect/authorize 請求參數，序列化成JSON暫存
    public DateTimeOffset ExpiresAt { get; set; }             // 短效，建議10分鐘（使用者在PNP登入頁花的時間要算進去）
    public bool Used { get; set; }                            // 單次使用，用過即標記，防重放
}
```

修改 AuthorizationServerDbContext.cs，加入這張表：
``` csharp
public class AuthorizationServerDbContext : DbContext
{
    public AuthorizationServerDbContext(DbContextOptions<AuthorizationServerDbContext> options)
        : base(options)
    {
    }

    public DbSet<PendingAuthorizeRequest> PendingAuthorizeRequests => Set<PendingAuthorizeRequest>();

    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);

        builder.Entity<PendingAuthorizeRequest>().HasKey(p => p.Nonce);

        builder.UseOpenIddict();
    }
}
```

說明：

- 這張表存的是短效、一次性的暫存資料，概念上跟設計文件第4節「Bridge一次性交換碼」很像，但這是不同的一組——那個存在PCBRepairNavi、用來證明「使用者在PNP登入有效」；這個存在Authorization Server自己這邊、單純是為了不要在跨站導來導去的過程中弄丟原始OAuth請求參數，兩者職責不同，不要搞混。
- OriginalRequestJson 先用JSON字串存，之後在Controller裡要用時反序列化回物件即可，不需要為每個欄位建獨立欄位。
- HasKey(p => p.Nonce) 明確指定Nonce為主鍵（因為它不是int/Guid的慣例命名Id，EF Core不會自動猜到，要手動指定）。

跑migration把這張表建出來：

``` bash
cd D:\Homer\Project\OAuth\OAuth.AuthorizationServer
dotnet ef migrations add AddPendingAuthorizeRequest
dotnet ef database update
```

請執行後確認：
1. dotnet build 正常編譯。
2. migration成功套用（做法跟之前Step 6一樣，看到Done.或資料庫裡查得到PendingAuthorizeRequests這張表）。
#
#
#
---

##   10：導向Bridge並正確繞回完成授權
這一步在做什麼：實作/connect/authorize收到請求後，第一次沒有身分時導去PCBRepairNavi的Bridge；Bridge驗證完、透過Redeem換到身分資料後，繞回/connect/authorize本身（不是另開一個端點）完成OpenIddict的授權簽發。這是正確版本，直接取代舊版「用獨立Resume端點直接簽發」的寫法（那個寫法在OpenIddict裡行不通，原因：OpenIddict只在請求路徑是設定的端點/connect/authorize時才會建立內部交易狀態，SignIn()只能在這個狀態存在時被正確攔截）。

10-1：appsettings.json 設定Bridge網址
``` json
{
  "ConnectionStrings": {
    "AuthorizationServerDb": "..."
  },
  "IdentityProviders": {
    "Pnp": {
      "BridgeUrl": "https://localhost:16888/sso/bridge"
    }
  }
}
```

10-2：PendingAuthorizeRequest.cs（在Step 9建立的基礎上，補上Bridge/Redeem結果欄位）
``` csharp
namespace OAuth.AuthorizationServer.Data;

public class PendingAuthorizeRequest
{
    public string Nonce { get; set; } = default!;
    public string OriginalRequestJson { get; set; } = default!;
    public DateTimeOffset ExpiresAt { get; set; }
    public bool Used { get; set; }

    // 由 BridgeCallback 填入，初始為 null
    public string? UserName { get; set; }
    public string? SiteCode { get; set; }
    public int? TokenId { get; set; }
    public DateTimeOffset? TicketExpiration { get; set; }
}
```

跑migration（如果Step 9之後已經跑過一次沒有這些欄位的版本，這裡會是追加一次新的migration）：

``` bash
cd D:\Homer\Project\OAuth\OAuth.AuthorizationServer
dotnet ef migrations add AddBridgeResultFieldsToPendingAuthorizeRequest
dotnet ef database update
```

10-3：Models\PendingAuthorizeData.cs
``` csharp
namespace OAuth.AuthorizationServer.Models;

public sealed class PendingAuthorizeData
{
    public string ClientId { get; set; } = default!;
    public string RedirectUri { get; set; } = default!;
    public string? Scope { get; set; }
    public string? State { get; set; }
    public string? CodeChallenge { get; set; }
    public string? CodeChallengeMethod { get; set; }
}
```

10-4：Controllers\AuthorizationController.cs
``` csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.WebUtilities;
using Microsoft.EntityFrameworkCore;
using OAuth.AuthorizationServer.Data;
using OAuth.AuthorizationServer.Models;
using OpenIddict.Abstractions;
using OpenIddict.Server.AspNetCore;
using System.Security.Claims;
using System.Security.Cryptography;
using System.Text.Json;
using static OpenIddict.Abstractions.OpenIddictConstants;

namespace OAuth.AuthorizationServer.Controllers;

public class AuthorizationController : Controller
{
    private readonly AuthorizationServerDbContext _db;
    private readonly IConfiguration _configuration;

    public AuthorizationController(AuthorizationServerDbContext db, IConfiguration configuration)
    {
        _db = db;
        _configuration = configuration;
    }

    [HttpGet("~/connect/authorize")]
    public async Task<IActionResult> Authorize([FromQuery(Name = "__nonce")] string? resumeNonce)
    {
        //雖然Authorize簽章沒有OAuth標準參數(client_id、response_type、redirect_uri、scope、code_challenge、code_challenge_method、state)，
        //但OAuth標準參數會由HttpContext.GetOpenIddictServerRequest()取得。沒給的話會報error!!
        var request = HttpContext.GetOpenIddictServerRequest()
            ?? throw new InvalidOperationException("OpenIddict 請求物件遺失。");

        // 第二次經過：Bridge/Redeem已完成，帶著 __nonce 繞回來
        if (!string.IsNullOrEmpty(resumeNonce))
        {
            return await CompleteAuthorizationAsync(resumeNonce, request);
        }

        // 第一次經過：還沒驗證身分，暫存原始請求，導去 PCBRepairNavi 的 Bridge
        var bridgeBaseUrl = _configuration["IdentityProviders:Pnp:BridgeUrl"]
            ?? throw new InvalidOperationException("找不到設定 'IdentityProviders:Pnp:BridgeUrl'");

        var nonce = GenerateNonce();

        var pending = new PendingAuthorizeRequest
        {
            Nonce = nonce,
            OriginalRequestJson = JsonSerializer.Serialize(new PendingAuthorizeData
            {
                ClientId = request.ClientId!,
                RedirectUri = request.RedirectUri!,
                Scope = request.Scope,
                State = request.State,
                CodeChallenge = request.CodeChallenge,
                CodeChallengeMethod = request.CodeChallengeMethod,
            }),
            ExpiresAt = DateTimeOffset.UtcNow.AddMinutes(10),
            Used = false,
        };

        _db.PendingAuthorizeRequests.Add(pending);
        await _db.SaveChangesAsync();

        var callbackUrl = Url.Action(nameof(BridgeCallback), "Authorization", values: null, protocol: Request.Scheme)!;
        var bridgeUrl = $"{bridgeBaseUrl}?return_url={Uri.EscapeDataString(callbackUrl)}&nonce={Uri.EscapeDataString(nonce)}";

        return Redirect(bridgeUrl);
    }

    /// <summary>
    /// PNP Callback
    /// </summary>
    /// <param name="bridgeCode">PNP產生，確認通過PNP登入驗證</param>
    /// <param name="nonce">Authorize產生</param>
    /// <returns></returns>
    [HttpGet("~/connect/authorize/bridge-callback")]
    public async Task<IActionResult> BridgeCallback([FromQuery] string bridgeCode, [FromQuery] string nonce)
    {
        var pending = await _db.PendingAuthorizeRequests.FirstOrDefaultAsync(p => p.Nonce == nonce);

        if (pending is null || pending.Used || pending.ExpiresAt < DateTimeOffset.UtcNow)
        {
            return BadRequest("無效或已過期的 nonce。");
        }

        //用bridgeCode換回PNP的身分資料{userName, siteCode, tokenId,ticketExpiration}
        var (userName, siteCode, tokenId, ticketExpiration) = await RedeemAsync(bridgeCode);

        pending.UserName = userName;
        pending.SiteCode = siteCode;
        pending.TokenId = tokenId;
        pending.TicketExpiration = ticketExpiration;
        // 注意：這裡先不標記 Used，等真正完成SignIn時才標記，避免Redeem成功但後面失敗時整筆被浪費

        await _db.SaveChangesAsync();

        var original = JsonSerializer.Deserialize<PendingAuthorizeData>(pending.OriginalRequestJson)!;

        var queryParams = new Dictionary<string, string?>
        {
            ["client_id"] = original.ClientId,
            ["redirect_uri"] = original.RedirectUri,
            ["response_type"] = "code",
            ["scope"] = original.Scope,
            ["state"] = original.State,
            ["code_challenge"] = original.CodeChallenge,
            ["code_challenge_method"] = original.CodeChallengeMethod,
            ["__nonce"] = nonce,
        };

        var authorizeUrl = QueryHelpers.AddQueryString("/connect/authorize",
            queryParams.Where(p => p.Value is not null).ToDictionary(p => p.Key, p => p.Value!));

        return Redirect(authorizeUrl);
    }

    private async Task<IActionResult> CompleteAuthorizationAsync(string resumeNonce, OpenIddictRequest request)
    {
        var pending = await _db.PendingAuthorizeRequests.FirstOrDefaultAsync(p => p.Nonce == resumeNonce);

        if (pending is null || pending.Used || pending.ExpiresAt < DateTimeOffset.UtcNow || pending.UserName is null)
        {
            return BadRequest("無效的授權狀態，請重新開始登入流程。");
        }

        pending.Used = true;
        await _db.SaveChangesAsync();

        var identity = new ClaimsIdentity(
            authenticationType: "PNP-Bridge",
            nameType: Claims.Name,
            roleType: Claims.Role);

        identity.SetClaim(Claims.Subject, pending.UserName);
        identity.SetClaim("site_code", pending.SiteCode);
        identity.SetClaim("token_id", pending.TokenId?.ToString());
        identity.SetClaim("ticket_expiration", pending.TicketExpiration?.ToUnixTimeSeconds().ToString());
        identity.SetScopes(request.GetScopes());

        var principal = new ClaimsPrincipal(identity);

        foreach (var claim in principal.Claims)
        {
            claim.SetDestinations(Destinations.AccessToken);
        }

        return SignIn(principal, OpenIddictServerAspNetCoreDefaults.AuthenticationScheme);
    }

    /// <summary>
    /// 從PNP取得身分驗證資料
    /// </summary>
    /// <param name="bridgeCode">PNP產生，確認通過PNP登入驗證</param>
    /// <returns></returns>
    private static Task<(string UserName, string SiteCode, int TokenId, DateTimeOffset TicketExpiration)> RedeemAsync(string bridgeCode)
    {
        // TODO（下一步）：改成真的呼叫 PCBRepairNavi 的 POST /api/internal/sso/redeem
        return Task.FromResult(("test\\mockuser", "AS", 1, DateTimeOffset.UtcNow.AddHours(12)));
    }

    private static string GenerateNonce()
    {
        var bytes = RandomNumberGenerator.GetBytes(32);
        return Convert.ToBase64String(bytes)
            .Replace('+', '-')
            .Replace('/', '_')
            .TrimEnd('=');
    }
}
```

幾個重點說明：

- HttpContext.GetOpenIddictServerRequest() 是OpenIddict提供的方法，因為Step 6開了EnableAuthorizationEndpointPassthrough()，這裡才能拿到OpenIddict已經先驗證過基本格式（client_id存在、redirect_uri吻合、PKCE必填等）的請求物件。
- GenerateNonce() 用 RandomNumberGenerator（密碼學安全的亂數來源，不是Random），產生URL-safe的字串。
- Authorize action身兼兩種情況：用resumeNonce（也就是__nonce查詢參數）是否存在來判斷「這是第一次經過還是第二次繞回來」。兩種情況都發生在同一個/connect/authorize路徑上，這是刻意的——OpenIddict只有在請求打到設定的端點路徑時，才會建立它內部需要的「交易狀態」，SignIn()只有在這個狀態存在時才能被正確攔截轉換成簽發Authorization Code的行為。如果在別的路徑（例如一個獨立的resume端點）呼叫SignIn()，OpenIddict會認不出這個請求、丟出例外。
- BridgeCallback本身不呼叫SignIn()：它的工作只有「驗證nonce、呼叫Redeem、把身分資料寫回PendingAuthorizeRequest、再轉一手導回/connect/authorize」。真正的SignIn()留到CompleteAuthorizationAsync（在Authorize第二次經過時才呼叫），原因同上。
- pending.Used在BridgeCallback裡先不標記true：故意的。如果標記了但後面CompleteAuthorizationAsync因為某些原因失敗，這筆暫存資料就變成「已用過但沒真的完成授權」的死資料，使用者會卡住。留到CompleteAuthorizationAsync真正成功簽完才標記，確保狀態一致。
- QueryHelpers.AddQueryString加上.Where(p => p.Value is not null)過濾：因為scope、state這些欄位在原始請求裡可能是null，直接塞進字典組URL會產生scope=這種空值參數，容易讓OpenIddict重新解析時行為不一致，所以先過濾掉null的。
- GenerateNonce()用RandomNumberGenerator而非Random：這個nonce是安全機制的一部分（防止有人用猜的nonce去冒用別人的Bridge結果），一定要用密碼學安全的亂數來源。
- identity.SetClaim(Claims.Subject, pending.UserName)等claims設定：Claims.Subject對應JWT的sub，site_code/token_id/ticket_expiration是我們自訂的claim，供之後Step 7（Refresh雙層檢查）讀取用；claim.SetDestinations(Destinations.AccessToken)明確指定這些claim要寫進access token裡，不設的話OpenIddict預設可能不會把自訂claim放進去。
- RedeemAsync目前回傳假資料：單純是因為PCBRepairNavi那邊的/api/internal/sso/redeem還沒做，先讓你能測完這條Authorization Server自己的邏輯，不用等對方。

請執行後確認：
1. dotnet build 正常編譯。
2. 用Insomnia依序測：
    1. https://localhost:7280/connect/authorize?client_id=pnp-mcp-client&response_type=code&redirect_uri=https://localhost:5055/callback&scope=ucs.sn-info.read&code_challenge=test&code_challenge_method=S256 → 預期302到bridge（連線失敗正常，PCBRepairNavi還沒做)
    2. https://localhost:7280/connect/bridge-callback?bridgeCode=TEST&nonce=<DB裡那筆nonce> → 預期302回/connect/authorize?...&__nonce=... 
    3. 步驟[2.2.]回傳的網址 → 預期這次簽發出Authorization Code、302到client的redirect_uri（連線失敗一樣正常，重點是有沒有拿到code參數）。
3. 查PendingAuthorizeRequests表，確認UserName等欄位有被填入、最後Used變成true。

最後整理一次執行順序：
MCP Client => Authenrize() => 302轉址到PNP的Bridge，進行登入確認流程，確定為登入後取得BridgeCode => BridgeCallback(bridgeCode, nonce) & RedeemAsync()取得PNP的身分驗證資料 => Authorize() => CompleteAuthorizationAsync()

1. MCP Client 發起請求（透過瀏覽器）→ 打到 Authorize()。
2. 因為沒帶__nonce（第一次經過），Authorize()存PendingAuthorizeRequest，回302 → 瀏覽器被導去PNP的Bridge（/sso/bridge，還沒做），PNP驗證完產生BridgeCode，再302導回我們的bridge-callback。
3. 瀏覽器打到 BridgeCallback(code, nonce)，裡面呼叫 RedeemAsync()（目前假資料）換到{userName, siteCode, tokenId, ticketExpiration}，寫回PendingAuthorizeRequest，再302導回/connect/authorize（帶__nonce）。
4. 瀏覽器再次打到 Authorize()（第二次經過，這次帶著__nonce）。
5. 因為帶了__nonce，呼叫 CompleteAuthorizationAsync()：<span style="color: red;">查出身分資料、組ClaimsPrincipal、SignIn()簽發Authorization Code</span>，302導回MCP Client真正的redirect_uri，流程結束。

<span style="color: red; font: bold">完成上述流程表示身分確認完成，從PNP取得了使用者資訊（userName/siteCode/tokenId/ticketExpiration）跟Authorizatiion Code，但還沒有取得AcessToken，我們必須要用Authorizatiion Code這個憑證，再去兌換AcessToken。</span>