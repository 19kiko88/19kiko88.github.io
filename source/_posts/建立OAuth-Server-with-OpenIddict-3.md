---
title: 建立OAuth-Server-with-OpenIddict-3
date: 2026-11-03 15:37:17
categories:
  - 後端開發
tags:
  - OAuth
  - OpenIddict
---
##### 接續上一篇...
<!-- more -->
#
#
#
---
## 建資料表，並把 client 綁定到身分來源，將來能當「共用身分驗證服務」，讓其他部門的系統也接進來。

這一步在做什麼：
設計文件的目標之一，是將來能當「共用身分驗證服務」，讓其他部門的系統也接進來。所以「要去哪裡驗證身分」不能寫死成 PNP，要變成一筆筆設定：

先看一下DB Table [OpenIddictApplications] 跟 [IdentityProvider] 這兩張Table的差異：
- OpenIddictApplications：<span style="color: red;">誰來要 token（OAuth Client）。</span>每個 OAuth Client 記錄自己屬於哪個身分來源(ex：MCP Client)。之後 /connect/authorize 看到 client_id，就知道要導去哪裡。
- IdentityProvider ：<span style="color: red;">使用者去哪裡驗證身分（身分來源）。</span>每一筆代表一個身分來源，記錄它的模式（Bridge、聯邦、直接驗證）和對接網址。目前只有 PNP 一筆。

{% asset_image table_compare.jpg table_compare %}

1. 新增檔案 Data/IdentityProvider.cs
``` csharp
namespace OAuth.AuthorizationServer.Data;

public class IdentityProvider
{
    // OpenIddict Application 的 Properties 裡，用這個 key 記錄該 Client 綁定哪個 IdentityProvider
    public const string ApplicationPropertyName = "identity_provider_id";

    public string ProviderId { get; set; } = default!;      // 主鍵，例如 "pnp"
    public IdentityProviderMode Mode { get; set; }          // 對接模式，見設計文件第6節
    public string? BridgeUrl { get; set; }                  // Bridge模式：身分來源的 /sso/bridge
    public string? RedeemApiUrl { get; set; }               // Bridge模式：用一次性code換回身分資料
    public string? RevocationCheckApiUrl { get; set; }      // refresh時查詢撤銷狀態（第4.3節第1層）
    public string? HmacSecretRef { get; set; }              // 指向金鑰管理機制的參照名稱，不存密鑰明文
}

public enum IdentityProviderMode
{
    /// <summary>
    /// 身分來源的登入狀態存在於其專屬 app 的 cookie/session 中，無法被 Authorization Server 跨網域讀取 | **PNP 採用此模式**
    /// </summary>
    Bridge,
    /// <summary>
    /// 身分來源已有公司既有的集中式 SSO（AD FS / Azure AD / Okta 等標準 OIDC/SAML） | 未來視接入對象而定
    /// </summary>
    Federation,  
    /// <summary>
    /// 身分來源為單純 AD/LDAP 帳密 | 未來視接入對象而定
    /// </summary>
    Direct,
}
```

2. Data/AuthorizationServerDbContext.cs：加上 DbSet 和設定
``` csharp
public DbSet<PendingAuthorizeRequest> PendingAuthorizeRequests => Set<PendingAuthorizeRequest>();
public DbSet<IdentityProvider> IdentityProviders => Set<IdentityProvider>();

protected override void OnModelCreating(ModelBuilder builder)
{
    base.OnModelCreating(builder);

    builder.Entity<PendingAuthorizeRequest>().HasKey(p => p.Nonce);

    builder.Entity<IdentityProvider>(e =>
    {
        e.HasKey(p => p.ProviderId);
        // enum 存成字串（"Bridge"），直接查DB時看得懂，日後調整enum順序也不會讓舊資料對錯值
        e.Property(p => p.Mode).HasConversion<string>();
    });

    // 讓 OpenIddict 把 Applications/Authorizations/Scopes/Tokens
    // 這四張表的 EF Core 模型註冊進這個 DbContext
    builder.UseOpenIddict();
}
```

3. Program.cs 的 SeedDataAsync：新增 PNP 身分來源，並讓 client 綁定它

最上面加上 using System.Text.Json;，然後在 const string mcpClientId = "pnp-mcp-client"; 前面加上：
``` csharp
var db = scope.ServiceProvider.GetRequiredService<AuthorizationServerDbContext>();
const string pnpProviderId = "pnp";

if (await db.IdentityProviders.FindAsync(pnpProviderId) is null)
{
    // TODO：網址為本機開發用，正式環境依PCBRepairNavi實際部署位置調整
    db.IdentityProviders.Add(new IdentityProvider
    {
        ProviderId = pnpProviderId,
        Mode = IdentityProviderMode.Bridge,
        BridgeUrl = "https://localhost:16888/sso/bridge",
        RedeemApiUrl = "https://localhost:16888/api/internal/sso/redeem",
        RevocationCheckApiUrl = "https://localhost:16888/api/internal/sso/token-status",
        HmacSecretRef = "IdentityProviders:Pnp:HmacSecret",
    });
    await db.SaveChangesAsync();
}
```

client 的部分，把原本的 if (await appManager.FindByClientIdAsync(mcpClientId) is null) { ... } 改成：
``` csharp
var application = await appManager.FindByClientIdAsync(mcpClientId);

if (application is null)
{
    var descriptor = new OpenIddictApplicationDescriptor
    {
        // ...（原本的 ClientId、ClientType、ConsentType、DisplayName、RedirectUris、Permissions 維持不變）
    };

    // ...（原本的 Permissions.Add 維持不變）

    // 綁定這個Client的身分來源
    descriptor.Properties[IdentityProvider.ApplicationPropertyName] = JsonSerializer.SerializeToElement(pnpProviderId);

    await appManager.CreateAsync(descriptor);
}
else if (!(await appManager.GetPropertiesAsync(application)).ContainsKey(IdentityProvider.ApplicationPropertyName))
{
    // 已存在但尚未綁定身分來源的Client（建立於這個欄位出現之前），補上綁定
    var descriptor = new OpenIddictApplicationDescriptor();
    await appManager.PopulateAsync(descriptor, application);
    descriptor.Properties[IdentityProvider.ApplicationPropertyName] = JsonSerializer.SerializeToElement(pnpProviderId);
    await appManager.UpdateAsync(application, descriptor);
}
```

4. 建立 migration 並套用到 DB

在 OAuth.AuthorizationServer 資料夾下執行：
``` bash
dotnet ef migrations add AddIdentityProvider
dotnet ef database update
```

幾個重點說明：

- <span style="color: red;">Client 綁定為什麼存在 Properties，而不是新增一個欄位：OpenIddictApplications 這張表的 entity 是 OpenIddict 套件定義的，不是我們的。要加欄位，必須自己繼承它的 entity 類別，再替換掉 OpenIddict 預設的 entity，改動很大。</span>OpenIddict 本來就提供 Properties 這個 JSON 欄位，專門讓使用者存自訂資料，用它最省事。DB 裡會看到 {"identity_provider_id":"pnp"}。
- JsonSerializer.SerializeToElement(...)：Properties 的值型別是 JsonElement，所以要先把字串 "pnp" 轉成 JsonElement 才能放進去。
- 為什麼 client 要多一個 else if 分支：你 DB 裡的 pnp-mcp-client 早就建好了。如果只改「不存在才新增」那段，它永遠不會被補上綁定。PopulateAsync 會把現有資料讀進 descriptor，加上綁定後再 UpdateAsync 寫回去，其他設定都不受影響。
- HmacSecretRef 存的是「參照名稱」，不是密鑰本身：設計文件第 9.1 節規定密鑰不能明文存在 DB 或設定檔。這裡只記「去哪裡拿」，例如將來的 KeyVault 名稱。真正讀取密鑰，是做 Redeem 那一步的事。
- RevocationCheckApiUrl 的路徑是暫定的：PNP 端的撤銷查詢端點還沒定名稱，先用 token-status 佔位。
- 這一步不會改變任何流程行為：Authorize() 還是讀 appsettings 的 BridgeUrl，下一步才會切換。

測試時要注意：這一步只需要看 DB。

1. dotnet ef migrations add 執行後，Migrations 資料夾多出 ..._AddIdentityProvider.cs。打開看一下，Up() 裡應該只有建立 IdentityProviders 表，不應該動到其他表。
2. dotnet ef database update 之後重啟 Server，到 DB 執行：
SELECT * FROM "IdentityProviders";
SELECT "ClientId", "Properties" FROM "OpenIddictApplications";

請執行後確認：
1. dotnet build 正常編譯，migration 的 Up() 只有建立 IdentityProviders 表。
2. IdentityProviders 有 1 筆 pnp，Mode 是字串 Bridge。
3. pnp-mcp-client 的 Properties 是 {"identity_provider_id":"pnp"}。

<span style="color: red;">總結一下這段在做什麼：</span>
OpenIddictApplications.Properties這個欄位可以拿來擴充OpenIddictApplications這個Table的欄位，不用異動到OpenIddict 套件定義好的 Entity。(先前有提到過)
我們可以用OpenIddictApplications.Properties的identity_provider_id的屬性值[pnp]來join到IdentityProviders.ProviderId

``` sql
SELECT a."ClientId", p."ProviderId", p."Mode", p."BridgeUrl", a."Properties" 
FROM "OpenIddictApplications" a
JOIN "IdentityProviders" p
ON p."ProviderId" = a."Properties"::jsonb ->> 'identity_provider_id';
```
{% asset_image json_join.jpg json_join %}

#
#
#
---
## /connect/authorize 改成依 client_id 去 DB 查身分來源，取代 appsettings 裡寫死的 BridgeUrl

這一步在做什麼：目前 Authorize() 第一次經過時，是讀 appsettings.json 的 IdentityProviders:Pnp:BridgeUrl。不管是哪個 client 來，都一律導去 PNP。這次改成上一則說的查詢順序：

client_id → OpenIddictApplications.Properties.identity_provider_id → IdentityProviders → BridgeUrl

改完之後，要新增身分來源或改 Bridge 網址，只要改 DB 資料，不用改程式。這就是設計文件第 6 節「核心協定邏輯不寫死任何特定身分來源」的意思。

1. AuthorizationController.cs：注入 IOpenIddictApplicationManager
``` csharp
private readonly IOpenIddictApplicationManager _applicationManager;

public AuthorizationController(
    AuthorizationServerDbContext db,
    IConfiguration configuration,
    ILogger<AuthorizationController> logger,
    IOpenIddictApplicationManager applicationManager,
    IOpenIddictScopeManager scopeManager,
    IOpenIddictAuthorizationManager authorizationManager,
    IOpenIddictTokenManager tokenManager)
{
    _db = db;
    _configuration = configuration;
    _logger = logger;
    _applicationManager = applicationManager;
    _scopeManager = scopeManager;
    _authorizationManager = authorizationManager;
    _tokenManager = tokenManager;
}
```

2. Authorize()：把讀 appsettings 的那段換掉
原本：
``` csharp
// 第一次經過：還沒驗證身分，暫存原始請求，導去 PCBRepairNavi 的 Bridge
var bridgeBaseUrl = _configuration["IdentityProviders:Pnp:BridgeUrl"]
    ?? throw new InvalidOperationException("找不到設定 'IdentityProviders:Pnp:BridgeUrl'");
```
改成：
``` csharp
// 第一次經過：還沒驗證身分，依 client_id 查出綁定的身分來源（設計文件第6節），決定要導去哪裡驗證
var provider = await ResolveIdentityProviderAsync(request.ClientId!);
if (provider is null)
{
    _logger.LogError("Authorize rejected. ClientId={ClientId}, Reason=IdentityProviderNotConfigured.", request.ClientId);
    return BadRequest("此 Client 未設定身分來源。");
}

// 目前只實作 Bridge 模式；聯邦/直接驗證模式待有實際接入對象時再實作
if (provider.Mode != IdentityProviderMode.Bridge || string.IsNullOrEmpty(provider.BridgeUrl))
{
    _logger.LogError(
        "Authorize rejected. ClientId={ClientId}, ProviderId={ProviderId}, Mode={Mode}, Reason=UnsupportedIdentityProvider.",
        request.ClientId, provider.ProviderId, provider.Mode);
    return BadRequest("此 Client 的身分來源設定不支援。");
}

var bridgeBaseUrl = provider.BridgeUrl;
```
下面 GenerateNonce()、建立 pending、組 bridgeUrl 的部分都維持不變。

3. 新增一個 private 方法
``` csharp
/// <summary>
/// 依 client_id 查出該 Client 綁定的身分來源：
/// OpenIddictApplications.Properties.identity_provider_id → IdentityProviders.ProviderId
/// </summary>
/// <returns>找不到綁定、或綁定的身分來源不存在時回傳 null</returns>
private async Task<IdentityProvider?> ResolveIdentityProviderAsync(string clientId)
{
    var application = await _applicationManager.FindByClientIdAsync(clientId);
    if (application is null)
    {
        return null;
    }

    var properties = await _applicationManager.GetPropertiesAsync(application);
    if (!properties.TryGetValue(IdentityProvider.ApplicationPropertyName, out var element)
        || element.ValueKind != JsonValueKind.String)
    {
        return null;
    }

    var providerId = element.GetString();
    return await _db.IdentityProviders.AsNoTracking().FirstOrDefaultAsync(p => p.ProviderId == providerId);
}
```

4. appsettings.json：刪掉不再使用的設定
把整個 "IdentityProviders": { "Pnp": { "BridgeUrl": ... } } 區塊刪掉。之後做 HMAC 那一步時，會再加回密鑰相關的設定。

幾個重點說明：

- ResolveIdentityProviderAsync 就是上一則那段 SQL join 的 C# 版本：先用 client_id 找到 client，從 Properties 讀出 identity_provider_id，再拿去查 IdentityProviders。因為沒有外鍵保護，每一關都要檢查查不到的情況，查不到就回傳 null。
- element.ValueKind != JsonValueKind.String：Properties 的值是 JsonElement，理論上可能被人存成數字或物件。先確認它是字串，再呼叫 GetString()，避免丟例外。
- 為什麼放在建立 pending 之前：設定有問題時直接拒絕，不會在 DB 留下一筆沒用的 pending。
- 為什麼用 LogError，不是之前常用的 LogWarning：之前的 LogWarning 大多是「請求有問題」，例如參數被竄改、nonce 過期，可能是攻擊或使用者操作問題。這次是「Server 設定有問題」，維運人員必須去修，所以等級較高。
- 為什麼 Mode 不是 Bridge 就拒絕：聯邦和直接驗證模式還沒實作。將來實作時，就在這裡依 provider.Mode 分流，導去不同的地方。
- IConfiguration 先保留：這一步之後 _configuration 暫時沒用到，但之後讀 HMAC 密鑰還會用到，所以先不刪。
- 還沒做的部分：這一步只讓「導去哪裡」改成依 DB 決定。bridge-callback 的 Redeem、refresh 的撤銷查詢也需要知道是哪個身分來源，那些留到「真正呼叫 PNP API」那一步一起改。

測試時要注意：第 2 項會暫時修改 DB 資料，測完記得改回來。

1. 確認導向的網址真的是從 DB 讀的：
   - 先打一次 /connect/authorize，確認還是 302 導向 https://localhost:16888/sso/bridge?...。
   - 到 DB 暫時把 Bridge 網址改成 16889：
UPDATE "IdentityProviders" SET "BridgeUrl" = 'https://localhost:16889/sso/bridge' WHERE "ProviderId" = 'pnp';
   - 不要重啟 Server，再打一次 /connect/authorize。
   - 改回來：
UPDATE "IdentityProviders" SET "BridgeUrl" = 'https://localhost:16888/sso/bridge' WHERE "ProviderId" = 'pnp';
2. 確認綁定錯誤時會被擋下：
   - 暫時把 client 綁定到一個不存在的身分來源：
UPDATE "OpenIddictApplications" SET "Properties" = '{"identity_provider_id":"nope"}' WHERE "ClientId" = 'pnp-mcp-client';
   - 重啟 Server 後打一次 /connect/authorize。<span style="color: red;">這裡要重啟，是因為 OpenIddict 會把 client 資料快取在記憶體裡，不重啟可能讀到舊值。</span>第 1 項的 IdentityProviders 是我們自己用 AsNoTracking 查的，沒有快取，所以不用重啟。
   - 改回來，再重啟一次：
UPDATE "OpenIddictApplications" SET "Properties" = '{"identity_provider_id":"pnp"}' WHERE "ClientId" = 'pnp-mcp-client';

請執行後確認：
1. dotnet build 正常編譯。
2. 第 1 項改成 16889 之後，不重啟就導向 https://localhost:16889/sso/bridge?...，證明網址是從 DB 讀的，不是 appsettings。
3. 第 2 項回 400「此 Client 未設定身分來源」，Console 有 Reason=IdentityProviderNotConfigured。
4. 兩項都改回來之後，/connect/authorize 恢復導向 16888。
#
#
#
---
## 讓「身分來源」跟著整個流程走，bridge-callback 和 refresh 都知道該找哪個身分來源

這一步在做什麼：上一步只有 Authorize() 第一次經過時會查身分來源。但流程中還有兩個地方，將來也要呼叫身分來源的 API：
{% asset_image call_api_method.jpg call_api_method %}

這兩個地方都拿不到 client_id 對應的 provider，所以要把「第一次經過時查到的 ProviderId」一路傳下去：

Authorize()           查到 provider → 存進 PendingAuthorizeRequest.ProviderId
    ↓
BridgeCallback()      從 pending 讀 ProviderId → 查 provider → 傳給 RedeemAsync
    ↓
CompleteAuthorization 從 pending 讀 ProviderId → 寫進 principal 的 "idp" claim
    ↓
Exchange (refresh)    從 refresh token 的 "idp" claim → 查 provider → 傳給 IsTokenActiveAsync

RedeemAsync 和 IsTokenActiveAsync 仍然是 mock，這一步只是把 provider 傳到它們手上。下一步改成真的呼叫時，直接用 provider.RedeemApiUrl、provider.RevocationCheckApiUrl 就好。

1. Data/PendingAuthorizeRequest.cs：新增欄位
``` csharp
public bool Used { get; set; }                            // 單次使用，用過即標記，防重放
public string ProviderId { get; set; } = default!;        // 第一次經過時依client_id查出的身分來源，bridge-callback要用它找RedeemApiUrl
```

2. AuthorizationController.cs：新增一個共用的查詢方法，並讓 ResolveIdentityProviderAsync 改用它
``` csharp
/// <summary>
/// 依 ProviderId 查出身分來源設定，找不到時回傳 null
/// </summary>
private Task<IdentityProvider?> FindIdentityProviderAsync(string? providerId)
{
    return _db.IdentityProviders.AsNoTracking().FirstOrDefaultAsync(p => p.ProviderId == providerId);
}
```
ResolveIdentityProviderAsync 最後一行改成：
``` csharp
return await FindIdentityProviderAsync(element.GetString());
```

3. Authorize()：建立 pending 時記下 ProviderId
``` csharp
ExpiresAt = DateTimeOffset.UtcNow.AddMinutes(10),
Used = false,
ProviderId = provider.ProviderId,
```

4. BridgeCallback()：在呼叫 RedeemAsync 之前查出 provider
``` csharp
// 依第一次經過時記下的身分來源，決定要去哪裡Redeem
var provider = await FindIdentityProviderAsync(pending.ProviderId);
if (provider is null)
{
    _logger.LogError("BridgeCallback rejected. ProviderId={ProviderId}, Reason=IdentityProviderNotFound.", pending.ProviderId);
    return BadRequest("身分來源設定不存在。");
}

//用bridgeCode換回PNP的身分資料{userName, siteCode, tokenId,ticketExpiration}
var (userName, siteCode, tokenId, ticketExpiration) = await RedeemAsync(provider, bridgeCode!);
```

RedeemAsync 的方法簽章改成：
``` csharp
/// <param name="provider">身分來源設定，之後要呼叫其 RedeemApiUrl</param>
/// <param name="bridgeCode">PNP產生，確認通過PNP登入驗證</param>
private static Task<(string UserName, string SiteCode, int TokenId, DateTimeOffset TicketExpiration)> RedeemAsync(IdentityProvider provider, string bridgeCode)
{
    // TODO（下一步）：改成真的呼叫 provider.RedeemApiUrl
    return Task.FromResult(("test\\mockuser", "AS", 1, DateTimeOffset.UtcNow.AddHours(12)));
}
```

5. CompleteAuthorizationAsync()：把 ProviderId 寫進 principal
``` csharp
identity.SetClaim(Claims.Subject, pending.UserName);
identity.SetClaim("idp", pending.ProviderId);
identity.SetClaim("site_code", pending.SiteCode);
```

6. Exchange() 的 refresh 分支：依 idp claim 查 provider
``` csharp
var userName = result.Principal.GetClaim(Claims.Subject);
var providerId = result.Principal.GetClaim("idp");
var tokenIdClaim = result.Principal.GetClaim("token_id");
var ticketExpirationClaim = result.Principal.GetClaim("ticket_expiration");

if (providerId is null || tokenIdClaim is null || ticketExpirationClaim is null)
{
    await RevokeTokenChainAsync(result.Principal);
    return RejectRefreshToken("缺少撤銷檢查所需的 claim。", userName);
}

// 依token所屬的身分來源，決定要去哪裡查撤銷狀態（設計文件第4.3節第1層）
var provider = await FindIdentityProviderAsync(providerId);
if (provider is null)
{
    await RevokeTokenChainAsync(result.Principal);
    return RejectRefreshToken("身分來源設定不存在。", userName);
}

// 第1層：IsTokenActive（目前mock，永遠回傳true；之後改成真的呼叫身分來源）
var isActive = await IsTokenActiveAsync(provider, int.Parse(tokenIdClaim));
```

IsTokenActiveAsync 的方法簽章改成：
``` csharp
private static Task<bool> IsTokenActiveAsync(IdentityProvider provider, int tokenId)
{
    // TODO：改成真的呼叫 provider.RevocationCheckApiUrl
    return Task.FromResult(true);
}
```

7. 建立 migration 並套用
``` bash
dotnet ef migrations add AddProviderIdToPendingAuthorizeRequest
dotnet ef database update
```

幾個重點說明：

- 為什麼 bridge-callback 存在 pending 裡，refresh 卻要用 claim：
  - bridge-callback 時，pending 還在，直接從 DB 讀就好。
  - refresh 時，pending 早就用完、被清掉了，手上只有 refresh token。所以要在簽發時把 ProviderId 寫進 principal，refresh token 解出來的 principal 才有。
- 為什麼 claim 叫 idp：這是業界常用的名稱，代表 Identity Provider，Azure AD、IdentityServer 都這樣用。它也會出現在 access token 裡，將來 Resource Server 也能看出這個使用者來自哪個身分來源。
- 為什麼 provider 查不到時也要撤銷整條鏈：這代表 token 所屬的身分來源已經被移除或設定錯誤，無法確認使用者是否還有效。依設計文件第 4.3 節，任一層檢查不通過，就要拒絕並撤銷。
- 舊的 refresh token 會被拒絕：這一步之前簽發的 refresh token 沒有 idp claim，refresh 時會走「缺少撤銷檢查所需的 claim」那條路被拒絕。這是預期行為，測試時要用改完之後新拿的 token。
- 抽出 FindIdentityProviderAsync：現在有 3 個地方需要依 ProviderId 查 provider，抽成一個方法，查詢邏輯只寫一次。

測試時要注意：用一次完整流程就能驗證所有改動。

請執行後確認：
1. dotnet build 正常編譯，migration 的 Up() 只新增 ProviderId 欄位。
2. 第 3 步那筆的 ProviderId 是 pnp。
3. <span style="color: red">access_token 的 payload 裡有 "idp": "pnp"。</span>
4. <span style="color: red">refresh 回 200，拿到的新 access_token 裡也有 "idp": "pnp"。</span>
#
#
#
---
## 建立 MockPnp 專案，模擬 PCBRepairNavi 的 /sso/bridge 和 Redeem API

這一步在做什麼：建一個小專案跑在 https://localhost:16888，代替還沒實作的 PNP 端。做完之後：
- /connect/authorize 導去 16888 時，終於有程式在接了。開著 Follow redirects，送一次 /connect/authorize 就會自動一路跑到拿到 code，不用再手動扮演 PNP。
- Redeem API 會照剛才定的格式驗證 HMAC 簽章，也會比對 bridgeCode 是不是為這個 nonce 發的，補上之前提過的綁定漏洞。
- 將來 PNP 團隊實作真正的端點時，可以拿這個專案當作 API 規格的參考。

這一步 Authorization Server 完全不用改，RedeemAsync 還是 mock。

1. 建立專案並加進方案

在 D:\Homer\Project\OAuth 下執行：
``` bash
dotnet new web -n OAuth.MockPnp -f net10.0
dotnet sln OAuth.slnx add OAuth.MockPnp
```

2. OAuth.MockPnp/Properties/launchSettings.json：把 https profile 的網址改成 16888
``` json
"https": {
  "commandName": "Project",
  "dotnetRunMessages": true,
  "launchBrowser": false,
  "applicationUrl": "https://localhost:16888",
  "environmentVariables": {
    "ASPNETCORE_ENVIRONMENT": "Development"
  }
}
```

3. OAuth.MockPnp/appsettings.json：加上 Sso 區塊
``` json
{
  "Sso": {
    "AllowedReturnUrls": [ "https://localhost:7280/connect/authorize/bridge-callback" ],
    "TicketLifetimeSeconds": 43200
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

4. 產生 HMAC 共用密鑰，存進 user-secrets

先產生一組隨機密鑰（Windows PowerShell 5.1 可用）：
``` bash
$b = New-Object byte[] 32; [Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($b); [Convert]::ToBase64String($b)
```

把輸出的字串記下來，下一步 Authorization Server 也要設定同一組。然後在 OAuth.MockPnp 資料夾下執行：
``` bash
dotnet user-secrets init
dotnet user-secrets set "Sso:HmacSecret" "<剛才產生的字串>"
```

5. OAuth.MockPnp/Program.cs：整個檔案換成下面這樣
``` csharp
using System.Collections.Concurrent;
using System.Security.Cryptography;
using System.Text;
using System.Text.Json;
using Microsoft.AspNetCore.WebUtilities;

// 模擬 PCBRepairNavi 的 SSO 端點，僅供本機開發測試 Authorization Server 使用。
// 真正的 PNP 端（.NET Framework 4.7.2）實作時，以這裡的 API 格式為準。
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

//HMAC，OAuth.AuthorizationServer跟OAuth.MockPnp 共用同一個 secret
var hmacSecret = builder.Configuration["Sso:HmacSecret"]
    ?? throw new InvalidOperationException("找不到設定 'Sso:HmacSecret'");
var allowedReturnUrls = builder.Configuration.GetSection("Sso:AllowedReturnUrls").Get<string[]>() ?? [];
var ticketLifetime = TimeSpan.FromSeconds(builder.Configuration.GetValue("Sso:TicketLifetimeSeconds", 43200));

// 一次性 bridge code，key 存 code 的 SHA256 而不是原值。真正的 PNP 端要存在 SQL Server（多主機共用）
var bridgeCodes = new ConcurrentDictionary<string, BridgeCodeEntry>();

app.MapGet("/sso/bridge", (string? return_url, string? nonce) =>
{
    // return_url 必須完全符合白名單（設計文件第4.1節）
    if (string.IsNullOrEmpty(return_url) || !allowedReturnUrls.Contains(return_url, StringComparer.Ordinal))
    {
        app.Logger.LogWarning("Bridge rejected. Reason=ReturnUrlNotAllowed, ReturnUrl={ReturnUrl}.", return_url);
        return Results.BadRequest("return_url 不在白名單內。");
    }

    if (string.IsNullOrEmpty(nonce))
    {
        return Results.BadRequest("缺少 nonce。");
    }

    // 真正的 PNP：FormsAuthentication.Decrypt(.ASPXAUTH) → IsTokenActive，沒登入就導去 /Account/Login
    // Mock：一律視為 test\mockuser 已登入
    var code = Base64Url(RandomNumberGenerator.GetBytes(32));
    bridgeCodes[Sha256Hex(code)] = new BridgeCodeEntry(
        Nonce: nonce,
        ExpiresAt: DateTimeOffset.UtcNow.AddSeconds(60),
        UserName: "test\\mockuser",
        SiteCode: "AS",
        TokenId: 1,
        TicketExpiration: DateTimeOffset.UtcNow.Add(ticketLifetime));

    app.Logger.LogInformation("Bridge succeeded. Nonce={Nonce}, bridge code issued.", nonce);

    return Results.Redirect(QueryHelpers.AddQueryString(return_url, new Dictionary<string, string?>
    {
        ["bridgeCode"] = code,
        ["nonce"] = nonce,
    }));
});

app.MapPost("/api/internal/sso/redeem", async (HttpRequest request) =>
{
    // 簽章要對「原始 body」計算，所以先讀成字串，驗完簽章才反序列化
    using var reader = new StreamReader(request.Body, Encoding.UTF8);
    var body = await reader.ReadToEndAsync();

    if (!VerifySignature(request, body))
    {
        return Results.Unauthorized();
    }

    var payload = JsonSerializer.Deserialize<RedeemRequest>(body, JsonSerializerOptions.Web);
    if (payload is null || string.IsNullOrEmpty(payload.Code) || string.IsNullOrEmpty(payload.Nonce))
    {
        return Results.BadRequest("缺少 code 或 nonce。");
    }

    // TryRemove：取出的同時刪除，同一個 code 只有一個請求拿得到（單次使用）。
    // 之後的檢查不論通過與否，這個 code 都已作廢
    if (!bridgeCodes.TryRemove(Sha256Hex(payload.Code), out var entry))
    {
        app.Logger.LogWarning("Redeem rejected. Reason=CodeNotFoundOrUsed.");
        return Results.BadRequest("code 無效或已使用。");
    }

    if (entry.ExpiresAt < DateTimeOffset.UtcNow)
    {
        app.Logger.LogWarning("Redeem rejected. Reason=CodeExpired.");
        return Results.BadRequest("code 已過期。");
    }

    // code 必須是為這個 nonce 發的，防止別人的 code 被接到這個登入流程上
    if (!string.Equals(entry.Nonce, payload.Nonce, StringComparison.Ordinal))
    {
        app.Logger.LogWarning("Redeem rejected. Reason=NonceMismatch.");
        return Results.BadRequest("nonce 不符。");
    }

    app.Logger.LogInformation("Redeem succeeded. UserName={UserName}.", entry.UserName);

    return Results.Ok(new RedeemResponse(
        entry.UserName, entry.SiteCode, entry.TokenId, entry.TicketExpiration.ToUnixTimeSeconds()));
});

app.Run();

// 驗證 X-Timestamp（±5秒）與 X-Signature（HMAC-SHA256）
bool VerifySignature(HttpRequest request, string body)
{
    if (!long.TryParse(request.Headers["X-Timestamp"], out var timestamp))
    {
        app.Logger.LogWarning("Signature rejected. Reason=MissingTimestamp.");
        return false;
    }

    if (Math.Abs(DateTimeOffset.UtcNow.ToUnixTimeSeconds() - timestamp) > 5)
    {
        app.Logger.LogWarning("Signature rejected. Reason=TimestampOutOfRange.");
        return false;
    }

    var expected = ComputeSignature(hmacSecret, timestamp, request.Method, request.Path, body);
    var actual = request.Headers["X-Signature"].ToString();

    // 用固定時間比對，避免攻擊者從回應時間差一個字元一個字元猜出簽章
    if (!CryptographicOperations.FixedTimeEquals(Encoding.ASCII.GetBytes(expected), Encoding.ASCII.GetBytes(actual)))
    {
        app.Logger.LogWarning("Signature rejected. Reason=InvalidSignature.");
        return false;
    }

    return true;
}

// 簽章內容：timestamp\nmethod\npath\nsha256(body)，兩端必須用完全相同的格式計算
static string ComputeSignature(string secret, long timestamp, string method, string path, string body)
{
    var canonical = $"{timestamp}\n{method}\n{path}\n{Sha256Hex(body)}";
    var signature = HMACSHA256.HashData(Encoding.UTF8.GetBytes(secret), Encoding.UTF8.GetBytes(canonical));
    return Convert.ToBase64String(signature);
}

static string Sha256Hex(string value) => Convert.ToHexStringLower(SHA256.HashData(Encoding.UTF8.GetBytes(value)));

static string Base64Url(byte[] bytes) => Convert.ToBase64String(bytes).Replace('+', '-').Replace('/', '_').TrimEnd('=');

record BridgeCodeEntry(string Nonce, DateTimeOffset ExpiresAt, string UserName, string SiteCode, int TokenId, DateTimeOffset TicketExpiration);
record RedeemRequest(string? Code, string? Nonce);
record RedeemResponse(string UserName, string SiteCode, int TokenId, long TicketExpiration);
```

幾個重點說明：

- /sso/bridge 跳過了真正的登入檢查：真正的 PNP 要解密 .ASPXAUTH、檢查 IsTokenActive，沒登入就導去登入頁。Mock 一律當作 test\mockuser 已經登入。要測試的是「Authorization Server 和 PNP 之間的對接」，不是 PNP 的登入本身。
- bridgeCode 只存雜湊值：Dictionary 的 key 是 SHA256(code)，不是 code 原文，這是照設計文件第 7 節「Code（hash）」的做法。萬一 DB 外洩，拿到的也只是雜湊值，不能拿來 Redeem。
- bridgeCode 和 nonce 綁在一起：發 code 時記下 nonce，Redeem 時要求兩者都送來，並且比對。這就是之前提過的「別人的 bridgeCode 接到這個登入流程上」的防護。
- 簽章為什麼要包含 timestamp、method、path、body 雜湊：
  - timestamp 加上 ±5 秒的限制：同一個請求被攔截後，過幾秒就不能再重送。
  - method、path：簽章不能被挪去打別的 API。
  - body 雜湊：body 被改過一個字元，簽章就對不上。
- 為什麼要先讀原始 body 再驗簽：簽章是對「送出的原始文字」算的。先反序列化再序列化回去，空白、欄位順序都可能改變，算出來的雜湊就會不同。
- CryptographicOperations.FixedTimeEquals：一般的字串比對遇到第一個不同的字元就停止，回應時間會因此有微小差異，攻擊者可能利用這個差異一個字元一個字元猜出簽章。固定時間比對可以避免這個問題。
- 密鑰為什麼放 user-secrets：設計文件第 9.1 節規定不能把密鑰明文寫在設定檔裡。user-secrets 存在你 Windows 使用者目錄下，不在專案資料夾，不會被 commit 進 git。正式環境改用 KeyVault 等方案。
- TicketLifetimeSeconds：預設 12 小時，跟真正的 PNP ticket 一樣。之後要測 refresh 的絕對上限時，改這個值就好，不用再改程式。
- Mock 的資料存在記憶體：MockPnp 重啟後，還沒 Redeem 的 code 就不見了。真正的 PNP 要存在 SQL Server，多台主機才能共用。

測試時要注意：兩個專案要同時跑。在各自的資料夾下執行 dotnet run --launch-profile https。

1. 自動跑完整條鏈：Follow redirects 開著，送一次 /connect/authorize（跟之前的網址一樣）。
2. 白名單：
https://localhost:16888/sso/bridge?return_url=https%3A%2F%2Fevil.example%2Fcb&nonce=test
3. 沒有簽章的 Redeem：用 POST 送 https://localhost:16888/api/internal/sso/redeem，body 選 JSON，內容填 {"code":"x","nonce":"y"}，不加任何 header。

請執行後確認：
1. 兩個專案都能正常啟動，MockPnp 顯示 Now listening on: https://localhost:16888。
2. 第 1 項送一次就跑到底，最後的 Location 是 https://localhost:5055/callback?code=...。MockPnp 的 Console 有 Bridge succeeded。目前 Authorization Server 還是 mock Redeem，所以 MockPnp 不會出現 Redeem succeeded，這是正常的。
{% asset_image mock_pnp_1.jpg mock_pnp_1 %}
成功呼叫Mock_Pnp api [/sso/bridge]
{% asset_image mock_pnp_2.jpg mock_pnp_2 %}

3. 第 2 項回 400「return_url 不在白名單內」。
4. 第 3 項回 401，MockPnp 的 Console 有 Reason=MissingTimestamp。

到目前為止。MockPnp 完成了：Bridge 能自動跑完整條鏈，白名單和簽章驗證也都正常擋下。<span style="color: red;">也代表從OAuth Server成功拿到了Authorization Code了。</span>
#
#
#
---
## Authorization Server 的 RedeemAsync 改成真的呼叫 Redeem API，並簽上 HMAC

這一步在做什麼：<span style="color: red;">目前 BridgeCallback() 拿到 bridgeCode 之後，呼叫的 RedeemAsync 是 mock，固定回傳 test\mockuser，完全沒有用到 bridgeCode。</span>這次改成真的照上一步定好的格式呼叫身分來源的 RedeemApiUrl：

POST {provider.RedeemApiUrl}
X-Timestamp: <現在的 unix 秒數>
X-Signature: HMAC-SHA256(密鑰, "timestamp\nPOST\npath\nsha256(body)")
{ "code": "<bridgeCode>", "nonce": "<nonce>" }

呼叫外部 API 的邏輯（組請求、簽章、讀密鑰）抽成一個獨立的 IdentityProviderClient 類別，不放在 Controller 裡。下一步做撤銷查詢時，也會用同一個類別和同一套簽章邏輯。

1. 把 HMAC 密鑰設進 Authorization Server 的 user-secrets<span style="color: red;">(上個步驟的HMAC，是存在MOCK_Pnp的，兩邊的HMAC需一致。)</span>

值要和 MockPnp 的完全一樣。可以先用 dotnet user-secrets list --project OAuth.MockPnp 查出來，然後在 D:\Homer\Project\OAuth 下執行：
``` bash
dotnet user-secrets init --project OAuth.AuthorizationServer
dotnet user-secrets set "IdentityProviders:Pnp:HmacSecret" "<和MockPnp相同的值>" --project OAuth.AuthorizationServer
```

可以在這先確認Mock_Pnp跟OAuth Server兩邊的HMAC是否一致
``` bash
dotnet user-secrets list --project OAuth.AuthorizationServer
dotnet user-secrets list --project OAuth.MockPnp
```

2. 新增檔案 OAuth.AuthorizationServer/Services/IdentityProviderClient.cs

``` csharp
using System.Security.Cryptography;
using System.Text;
using System.Text.Json;
using OAuth.AuthorizationServer.Data;

namespace OAuth.AuthorizationServer.Services;

/// <summary>
/// 呼叫身分來源（如PCBRepairNavi）的內部API：組請求、附上HMAC簽章（設計文件第4.2節）
/// </summary>
public class IdentityProviderClient
{
    private readonly HttpClient _httpClient;
    private readonly IConfiguration _configuration;
    private readonly ILogger<IdentityProviderClient> _logger;

    public IdentityProviderClient(
        HttpClient httpClient,
        IConfiguration configuration,
        ILogger<IdentityProviderClient> logger)
    {
        _httpClient = httpClient;
        _configuration = configuration;
        _logger = logger;
    }

    /// <summary>
    /// 用一次性 bridgeCode 換回身分資料
    /// </summary>
    /// <returns>Redeem 失敗（簽章錯誤、code無效、網路錯誤等）時回傳 null</returns>
    public async Task<RedeemResult?> RedeemAsync(IdentityProvider provider, string bridgeCode, string nonce)
    {
        var body = JsonSerializer.Serialize(new { code = bridgeCode, nonce }, JsonSerializerOptions.Web);
        using var request = CreateSignedRequest(provider, HttpMethod.Post, provider.RedeemApiUrl, body);

        try
        {
            using var response = await _httpClient.SendAsync(request);
            if (!response.IsSuccessStatusCode)
            {
                _logger.LogWarning("Redeem failed. ProviderId={ProviderId}, StatusCode={StatusCode}.",
                    provider.ProviderId, (int)response.StatusCode);
                return null;
            }

            var result = await response.Content.ReadFromJsonAsync<RedeemResult>(JsonSerializerOptions.Web);
            if (result is null || string.IsNullOrEmpty(result.UserName))
            {
                _logger.LogWarning("Redeem failed. ProviderId={ProviderId}, Reason=InvalidResponse.", provider.ProviderId);
                return null;
            }

            return result;
        }
        catch (Exception ex) when (ex is HttpRequestException or TaskCanceledException)
        {
            // 連不上身分來源或逾時
            _logger.LogError(ex, "Redeem failed. ProviderId={ProviderId}, Reason=RequestError.", provider.ProviderId);
            return null;
        }
    }

    /// <summary>
    /// 建立附有 X-Timestamp、X-Signature 的請求
    /// </summary>
    private HttpRequestMessage CreateSignedRequest(IdentityProvider provider, HttpMethod method, string? url, string body)
    {
        if (!Uri.TryCreate(url, UriKind.Absolute, out var uri) || uri.Scheme != Uri.UriSchemeHttps)
        {
            // 設計文件第4.2節：僅接受HTTPS
            throw new InvalidOperationException($"身分來源 '{provider.ProviderId}' 的 API 網址未設定或不是 HTTPS。");
        }

        // HmacSecretRef 是設定的 key 名稱，實際密鑰從設定讀取（開發環境：user-secrets；正式環境：金鑰管理機制）
        var secret = string.IsNullOrEmpty(provider.HmacSecretRef) ? null : _configuration[provider.HmacSecretRef];
        if (string.IsNullOrEmpty(secret))
        {
            throw new InvalidOperationException($"找不到身分來源 '{provider.ProviderId}' 的 HMAC 密鑰設定。");
        }

        var timestamp = DateTimeOffset.UtcNow.ToUnixTimeSeconds();

        var request = new HttpRequestMessage(method, uri)
        {
            Content = new StringContent(body, Encoding.UTF8, "application/json"),
        };
        request.Headers.Add("X-Timestamp", timestamp.ToString());
        request.Headers.Add("X-Signature", ComputeSignature(secret, timestamp, method.Method, uri.AbsolutePath, body));

        return request;
    }

    // 簽章內容：timestamp\nmethod\npath\nsha256(body)，必須與身分來源端的驗證格式完全一致
    private static string ComputeSignature(string secret, long timestamp, string method, string path, string body)
    {
        var bodyHash = Convert.ToHexStringLower(SHA256.HashData(Encoding.UTF8.GetBytes(body)));
        var canonical = $"{timestamp}\n{method}\n{path}\n{bodyHash}";
        var signature = HMACSHA256.HashData(Encoding.UTF8.GetBytes(secret), Encoding.UTF8.GetBytes(canonical));
        return Convert.ToBase64String(signature);
    }
}

public record RedeemResult(string UserName, string SiteCode, int TokenId, long TicketExpiration);
```

3. Program.cs：註冊 IdentityProviderClient

在 AddHostedService<PendingAuthorizeRequestCleanupService>() 下面加上：
``` csharp
// 呼叫身分來源內部API（Redeem、撤銷查詢）用的HttpClient
builder.Services.AddHttpClient<IdentityProviderClient>(client => client.Timeout = TimeSpan.FromSeconds(10));
```

4. AuthorizationController.cs：改用 IdentityProviderClient

建構子把不再使用的 IConfiguration 換成 IdentityProviderClient：
``` csharp
private readonly AuthorizationServerDbContext _db;
private readonly ILogger<AuthorizationController> _logger;
private readonly IdentityProviderClient _identityProviderClient;
// ...（其他 manager 欄位維持不變）

public AuthorizationController(
    AuthorizationServerDbContext db,
    ILogger<AuthorizationController> logger,
    IdentityProviderClient identityProviderClient,
    IOpenIddictApplicationManager applicationManager,
    IOpenIddictScopeManager scopeManager,
    IOpenIddictAuthorizationManager authorizationManager,
    IOpenIddictTokenManager tokenManager)
{
    _db = db;
    _logger = logger;
    _identityProviderClient = identityProviderClient;
    // ...（其他 manager 的指派維持不變）
}
```

檔案最上面要加 using OAuth.AuthorizationServer.Services;。

BridgeCallback() 裡原本呼叫 mock 的那一行：
``` csharp
var (userName, siteCode, tokenId, ticketExpiration) = await RedeemAsync(provider, bridgeCode!);
```

換成：
``` csharp
//用bridgeCode換回身分資料{userName, siteCode, tokenId, ticketExpiration}
var redeem = await _identityProviderClient.RedeemAsync(provider, bridgeCode!, nonce!);
if (redeem is null)
{
    _logger.LogWarning("BridgeCallback rejected. ProviderId={ProviderId}, Reason=RedeemFailed.", provider.ProviderId);
    return BadRequest("身分驗證失敗，請重新登入。");
}

var (userName, siteCode, tokenId, ticketExpirationUnix) = redeem;
var ticketExpiration = DateTimeOffset.FromUnixTimeSeconds(ticketExpirationUnix);
```
最後把舊的 private static Task<...> RedeemAsync(IdentityProvider provider, string bridgeCode) 整個方法刪掉。

幾個重點說明：

- AddHttpClient<IdentityProviderClient>：這是 ASP.NET Core 建議的 HttpClient 用法（typed client）。框架會自動建立、重複使用底層連線，再注入 IdentityProviderClient。不建議自己 new HttpClient()，那樣用久了可能把連線用光。
- Timeout = 10 秒：PNP 沒有回應時，最多等 10 秒就放棄。否則使用者的瀏覽器會一直轉圈。
- 為什麼 nonce 也要送：這就是上一步 MockPnp 檢查的 NonceMismatch。PNP 會確認這個 bridgeCode 是為這個 nonce 發的。
- 簽章用 uri.AbsolutePath：只取路徑，例如 /api/internal/sso/redeem，不含主機名稱和 query string，和 MockPnp 那邊的 request.Path 一致。兩邊用來簽章的內容只要差一個字元，簽章就對不上。
- Redeem 失敗時回傳 null，不丟例外：簽章錯誤、code 無效、連不上 PNP，都是「這次登入失敗」，由 Controller 回 400 給使用者重新登入就好。只有「設定錯誤」（網址不是 HTTPS、找不到密鑰）才丟例外，因為那需要維運人員修正，不是使用者重試能解決的。
- Redeem 失敗時 pending 不作廢：pending 還沒被回填，10 分鐘內可以重新走一次 PNP。現在 PNP 會比對 bridgeCode 和 nonce，攻擊者沒辦法用別人的 code 搭上這個 nonce。
- var (userName, ...) = redeem;：record 可以直接解構成多個變數，後面 ExecuteUpdateAsync 用到的變數名稱和型別都和原本一樣，所以那段不用改。

測試時要注意：兩個專案都要跑，Follow redirects 開著。

1. 正常流程：送一次 /connect/authorize，從最後的 Location 複製 code，再到 /connect/token 換 token。
2. 密鑰錯誤：
   - 把 Authorization Server 的密鑰暫時改成錯的值：
dotnet user-secrets set "IdentityProviders:Pnp:HmacSecret" "wrong-secret" --project OAuth.AuthorizationServer
   - 重啟 Authorization Server，再送一次 /connect/authorize。
   - 測完改回正確的值，再重啟。

請執行後確認：
1. dotnet build 正常編譯。
2. 第 1 項換到 token，MockPnp 的 Console 這次會出現 Redeem succeeded. UserName=test\mockuser，代表 Authorization Server 真的去呼叫了 Redeem API。
{% asset_image mock_pnp_3.jpg mock_pnp_3 %}

3. 第 2 項最後停在 bridge-callback，回 400「身分驗證失敗，請重新登入」：
   - MockPnp 的 Console 有 Reason=InvalidSignature。
   - Authorization Server 的 Console 有 Redeem failed ... StatusCode=401 和 Reason=RedeemFailed。

**到此步驟為止：Redeem 已經改成真的呼叫 PNP，HMAC 兩端也對得上。**
#
#
#
---
## 撤銷查詢（IsTokenActiveAsync）改成真的呼叫身分來源

這一步在做什麼：refresh 的第 1 層檢查（設計文件第 4.3 節）是要問 PNP「這個 tokenId 的使用者是不是已經登出或被踢了」，但目前 IsTokenActiveAsync 是 mock，永遠回傳 true，所以這一層形同虛設。這次兩邊一起改：

- MockPnp：新增撤銷查詢 API token-status，用同一套 HMAC 驗證。另外加一個只供測試用的 /dev/logout/{tokenId}，模擬「使用者在 PNP 登出或被踢」。每次 Bridge 也改成發一個新的 tokenId，模擬真正 PNP 每次登入都有自己的 UserToken。
- Authorization Server：IdentityProviderClient 新增 IsTokenActiveAsync，refresh 時改呼叫它。

撤銷查詢 API 的格式，比照 Redeem：
POST https://localhost:16888/api/internal/sso/token-status
X-Timestamp: 1793180718
X-Signature: （和 Redeem 同一套算法）
Content-Type: application/json

{"tokenId":1}

→ 200 {"active":true}

1. MockPnp 的 Program.cs

在 var bridgeCodes = ... 下面加上：
``` csharp
// 模擬 PNP 的 UserToken：每次登入發一個新的 tokenId；被登出/被踢的 tokenId 記在這裡
var nextTokenId = 0;
var revokedTokenIds = new ConcurrentDictionary<int, bool>();

// /sso/bridge 裡建立 BridgeCodeEntry 那段，把 TokenId: 1 改掉，log 也一起帶出 tokenId：
var tokenId = Interlocked.Increment(ref nextTokenId);
var code = Base64Url(RandomNumberGenerator.GetBytes(32));
bridgeCodes[Sha256Hex(code)] = new BridgeCodeEntry(
    Nonce: nonce,
    ExpiresAt: DateTimeOffset.UtcNow.AddSeconds(60),
    UserName: "test\\mockuser",
    SiteCode: "AS",
    TokenId: tokenId,
    TicketExpiration: DateTimeOffset.UtcNow.Add(ticketLifetime));

app.Logger.LogInformation("Bridge succeeded. Nonce={Nonce}, TokenId={TokenId}, bridge code issued.", nonce, tokenId);
```

在 app.Run(); 前面加上兩個端點：
``` csharp
app.MapPost("/api/internal/sso/token-status", async (HttpRequest request) =>
{
    using var reader = new StreamReader(request.Body, Encoding.UTF8);
    var body = await reader.ReadToEndAsync();

    if (!VerifySignature(request, body))
    {
        return Results.Unauthorized();
    }

    var payload = JsonSerializer.Deserialize<TokenStatusRequest>(body, JsonSerializerOptions.Web);
    if (payload is null)
    {
        return Results.BadRequest("缺少 tokenId。");
    }

    // 真正的 PNP：MSSQLDataContext.IsTokenActive(tokenId)，即 UserToken.LogoutTime IS NULL
    var active = !revokedTokenIds.ContainsKey(payload.TokenId);
    app.Logger.LogInformation("TokenStatus checked. TokenId={TokenId}, Active={Active}.", payload.TokenId, active);

    return Results.Ok(new TokenStatusResponse(active));
});

// 僅供本機測試：模擬使用者在 PNP 登出或被踢（真正的 PNP 沒有這個端點）
app.MapPost("/dev/logout/{tokenId:int}", (int tokenId) =>
{
    revokedTokenIds[tokenId] = true;
    app.Logger.LogInformation("Dev logout. TokenId={TokenId} marked as logged out.", tokenId);
    return Results.Ok($"tokenId {tokenId} 已標記為登出。");
});
```

檔案最下面的 record 加上兩個：
``` csharp
record TokenStatusRequest(int TokenId);
record TokenStatusResponse(bool Active);
```

2. Authorization Server 的 Services/IdentityProviderClient.cs
在 RedeemAsync 下面新增方法：
``` csharp
/// <summary>
/// 查詢身分來源端的登入狀態是否仍有效（設計文件第4.3節第1層）
/// </summary>
/// <returns>仍有效回傳 true；已登出/被踢，或查詢失敗（連不上、簽章錯誤等）一律回傳 false</returns>
public async Task<bool> IsTokenActiveAsync(IdentityProvider provider, int tokenId)
{
    var body = JsonSerializer.Serialize(new { tokenId }, JsonSerializerOptions.Web);
    using var request = CreateSignedRequest(provider, HttpMethod.Post, provider.RevocationCheckApiUrl, body);

    try
    {
        using var response = await _httpClient.SendAsync(request);
        if (!response.IsSuccessStatusCode)
        {
            _logger.LogWarning("TokenStatus check failed. ProviderId={ProviderId}, TokenId={TokenId}, StatusCode={StatusCode}.",
                provider.ProviderId, tokenId, (int)response.StatusCode);
            return false;
        }

        var result = await response.Content.ReadFromJsonAsync<TokenStatusResult>(JsonSerializerOptions.Web);
        return result?.Active ?? false;
    }
    catch (Exception ex) when (ex is HttpRequestException or TaskCanceledException)
    {
        _logger.LogError(ex, "TokenStatus check failed. ProviderId={ProviderId}, TokenId={TokenId}, Reason=RequestError.",
            provider.ProviderId, tokenId);
        return false;
    }
}
```

檔案最下面的 record 加上：
``` csharp
public record TokenStatusResult(bool Active);
```

3. AuthorizationController.cs
refresh 分支裡的第 1 層檢查改成：
``` csharp
// 第1層：向身分來源查詢 tokenId 是否已被登出/被踢（查詢失敗也視為未通過）
var isActive = await _identityProviderClient.IsTokenActiveAsync(provider, int.Parse(tokenIdClaim));
```

然後把舊的 private static Task<bool> IsTokenActiveAsync(IdentityProvider provider, int tokenId) 整個方法刪掉。

幾個重點說明：

- 查詢失敗為什麼也回傳 false：這叫 fail closed（失敗時採取保守做法）。連不上 PNP，就無法確認使用者是不是已經被踢了，所以寧可拒絕。反過來的話，攻擊者只要讓 PNP 暫時連不上，就能繞過撤銷檢查。代價是 PNP 故障期間 refresh 都會失敗，使用者要重新登入。不過 PNP 故障時，重新登入本來就需要 PNP，所以這個代價本來就無法避免。
- Interlocked.Increment(ref nextTokenId)：同時有多個請求進來時，保證每個人拿到的 tokenId 都不同。直接寫 nextTokenId++ 在並行時可能發出重複的號碼。
- /dev/logout 不需要簽章：它只是測試工具，代替你去 PNP 網站按登出，真正的 PNP 不會有這個端點。它放在 MockPnp 裡，不會出現在 Authorization Server。
- MockPnp 重啟後，登出紀錄和 tokenId 計數都會歸零：資料都存在記憶體。測試時如果中途重啟 MockPnp，要重新跑一次完整流程。
- Refresh 之外，什麼時候會檢查撤銷：只有 refresh 時才會查。已經發出去的 access_token 在 30 分鐘到期前仍然有效，這是設計文件第 9.3 節接受的取捨：權限變更在下次 refresh 時才生效。

測試時要注意：兩個專案都改完後都要重啟，Follow redirects 開著。

1. 正常 refresh：跑一次完整流程，scope 含 offline_access，拿到 token 之後解開 access_token，看 token_id 是多少。再用 refresh_token 換一次。
2. 模擬被踢：用 POST 送 https://localhost:16888/dev/logout/<上一步的 token_id>，然後用最新的 refresh_token 再換一次。
    - 先依照正常流程來取得acess_token跟refresh_token
    - 解開acess_token，取得token_id(4)
    {% asset_image logout_test_1.jpg logout_test_1 %}
    - 執行api [dev/logout] https://localhost:16888/dev/logout/<token_id>。
    {% asset_image logout_test_2.jpg logout_test_2 %}
    - 用步驟1.的refresh_token在執行一次 API [connect/token](grant_type為refresh_token)。
    {% asset_image logout_test_3.jpg logout_test_3 %}

請執行後確認：
1. 兩個專案都能正常編譯、啟動。
2. 第 1 項的 refresh 回 200，MockPnp 的 Console 有 TokenStatus checked. TokenId=N, Active=True。
3. 第 2 項的 refresh 回 400 invalid_grant，error_description 是「PNP tokenId 已被登出/被踢。」。MockPnp 有 Active=False，Authorization Server 有 RevokeTokenChain completed。

**MockPnp**
{% asset_image logout_test_4.jpg logout_test_4 %}

**Authorization Server**
{% asset_image logout_test_5.jpg logout_test_5 %}