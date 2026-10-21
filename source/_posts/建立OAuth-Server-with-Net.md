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