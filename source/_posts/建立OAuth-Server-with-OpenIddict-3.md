---
title: 建立OAuth-Server-with-OpenIddict(3)
date: 2026-11-03 15:37:17
categories:
  - 後端開發
tags:
  - OAuth
  - OpenIddict
---
##### 接續上一篇...
##### 此篇重點：把驗證到取得token的流程串接起來
- 建資料表，並把 client 綁定到身分來源，將來能當「共用身分驗證服務」，讓其他部門的系統也接進來。
- /connect/authorize 改成依 client_id 去 DB 查身分來源，取代 appsettings 裡寫死的 BridgeUrl
- 讓「身分來源」跟著整個流程走，bridge-callback 和 refresh 都知道該找哪個身分來源
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