---
title: 建立OAuth Server with OpenIddict(8)
date: 2026-11-18 17:25:20
categories:
  - 後端開發
tags:
  - OAuth
  - OpenIddict
---
##### 接續上一篇...
##### 把 AuthorizationController 的 DB 邏輯抽到 OAuth.Core
<!-- more -->
#
#
#
---
這一步在做什麼：Controller 目前大約 480 行，同時處理「OAuth 流程判斷」和「DB 怎麼讀寫」。這次把 DB 的部分抽成兩個類別，放在 OAuth.Core/Stores/：
{% asset_image 重構AuthorizationController.jpg 重構AuthorizationController %}

搬完之後，Controller 只剩 OAuth 流程的判斷，不再直接碰 DbContext。這次是純搬移，行為完全不變，所以 30 個測試應該全部照樣通過。另外加 4 個直接測 Store 的測試，示範抽出來之後可以單獨測 DB 邏輯。

搬移時順便清掉兩個舊的小問題：
- CompleteAuthorizationAsync 裡的 pending.Used = true; await _db.SaveChangesAsync(); 是改成原子性更新之前留下的。pending 是用 AsNoTracking() 讀出來的，這兩行其實什麼都沒做。
- BridgeCallback 裡 AddQueryString(... p => p.Value!) 的可 null 警告。
#
#
#
---
## 把 AuthorizationController 的 DB 邏輯抽到 OAuth.Core

1. 把 PendingAuthorizeData 移到 OAuth.Core
Store 需要用到它，所以從網站的 Models/ 移到 OAuth.Core/Data/：
git mv OAuth.AuthorizationServer/Models/PendingAuthorizeData.cs OAuth.Core/Data/PendingAuthorizeData.cs
打開 OAuth.Core/Data/PendingAuthorizeData.cs，把 namespace OAuth.AuthorizationServer.Models; 改成 namespace OAuth.Core.Data;。Models 資料夾空了，可以刪掉。

2. 新增 OAuth.Core/Stores/PendingAuthorizeRequestStore.cs
``` csharp
using System.Security.Cryptography;
using System.Text.Json;
using Microsoft.EntityFrameworkCore;
using OAuth.Core.Data;

namespace OAuth.Core.Stores;

/// <summary>
/// PendingAuthorizeRequest 的資料存取（設計文件第4.1節）。
/// 「回填身分」與「取得使用權」都以單一條 UPDATE 完成，確保並行請求下仍只會成功一次
/// </summary>
public class PendingAuthorizeRequestStore
{
    private static readonly TimeSpan Lifetime = TimeSpan.FromMinutes(10);

    private readonly AuthorizationServerDbContext _db;

    public PendingAuthorizeRequestStore(AuthorizationServerDbContext db)
    {
        _db = db;
    }

    /// <summary>
    /// 第一次經過 /connect/authorize：暫存原始請求參數，回傳產生的 nonce
    /// </summary>
    public async Task<string> CreateAsync(PendingAuthorizeData original, string providerId)
    {
        var nonce = GenerateNonce();

        _db.PendingAuthorizeRequests.Add(new PendingAuthorizeRequest
        {
            Nonce = nonce,
            OriginalRequestJson = JsonSerializer.Serialize(original),
            ExpiresAt = DateTimeOffset.UtcNow.Add(Lifetime),
            Used = false,
            ProviderId = providerId,
        });
        await _db.SaveChangesAsync();

        return nonce;
    }

    /// <summary>
    /// bridge-callback：取得仍可回填身分的 pending（存在、未使用、未過期、尚未回填），否則回傳 null
    /// </summary>
    public Task<PendingAuthorizeRequest?> FindFillableAsync(string nonce)
    {
        var now = DateTimeOffset.UtcNow;
        return _db.PendingAuthorizeRequests.AsNoTracking()
            .FirstOrDefaultAsync(p => p.Nonce == nonce && !p.Used && p.ExpiresAt > now && p.UserName == null);
    }

    /// <summary>
    /// bridge-callback：以單一條 UPDATE 原子性地「確認尚未回填 + 寫入身分資料」，同一個 nonce 只有第一次會成功。
    /// 這裡先不標記 Used，等真正完成 SignIn 時才標記，避免 Redeem 成功但後面失敗時整筆被浪費
    /// </summary>
    public async Task<bool> TryFillIdentityAsync(string nonce, string userName, string siteCode, int tokenId, DateTimeOffset ticketExpiration)
    {
        var now = DateTimeOffset.UtcNow;
        var filled = await _db.PendingAuthorizeRequests
            .Where(p => p.Nonce == nonce && !p.Used && p.ExpiresAt > now && p.UserName == null)
            .ExecuteUpdateAsync(s => s
                .SetProperty(p => p.UserName, userName)
                .SetProperty(p => p.SiteCode, siteCode)
                .SetProperty(p => p.TokenId, (int?)tokenId)
                .SetProperty(p => p.TicketExpiration, (DateTimeOffset?)ticketExpiration));

        return filled == 1;
    }

    /// <summary>
    /// 第二次經過 /connect/authorize：以單一條 UPDATE 原子性地「檢查有效 + 標記已使用」，
    /// 同一個 nonce 同時被送多次時只有一個請求會成功。成功回傳該筆資料（此時 Used 已是 true），否則回傳 null
    /// </summary>
    public async Task<PendingAuthorizeRequest?> TryConsumeAsync(string nonce)
    {
        var now = DateTimeOffset.UtcNow;
        var consumed = await _db.PendingAuthorizeRequests
            .Where(p => p.Nonce == nonce && !p.Used && p.ExpiresAt > now && p.UserName != null)
            .ExecuteUpdateAsync(s => s.SetProperty(p => p.Used, true));

        if (consumed == 0)
        {
            return null;
        }

        return await _db.PendingAuthorizeRequests.AsNoTracking().FirstAsync(p => p.Nonce == nonce);
    }

    /// <summary>
    /// 刪除已過期的資料，回傳刪除筆數
    /// </summary>
    public Task<int> DeleteExpiredAsync(CancellationToken cancellationToken = default)
    {
        var now = DateTimeOffset.UtcNow;
        return _db.PendingAuthorizeRequests
            .Where(p => p.ExpiresAt < now)
            .ExecuteDeleteAsync(cancellationToken);
    }

    /// <summary>
    /// 讀出第一次經過時暫存的原始請求參數
    /// </summary>
    public static PendingAuthorizeData ReadOriginalRequest(PendingAuthorizeRequest pending)
    {
        return JsonSerializer.Deserialize<PendingAuthorizeData>(pending.OriginalRequestJson)!;
    }

    private static string GenerateNonce()
    {
        return Convert.ToBase64String(RandomNumberGenerator.GetBytes(32))
            .Replace('+', '-')
            .Replace('/', '_')
            .TrimEnd('=');
    }
}
```

3. 新增 OAuth.Core/Stores/IdentityProviderStore.cs
``` csharp
using System.Text.Json;
using Microsoft.EntityFrameworkCore;
using OAuth.Core.Data;
using OpenIddict.Abstractions;

namespace OAuth.Core.Stores;

/// <summary>
/// 身分來源的查詢（設計文件第6節）
/// </summary>
public class IdentityProviderStore
{
    private readonly AuthorizationServerDbContext _db;
    private readonly IOpenIddictApplicationManager _applicationManager;

    public IdentityProviderStore(AuthorizationServerDbContext db, IOpenIddictApplicationManager applicationManager)
    {
        _db = db;
        _applicationManager = applicationManager;
    }

    /// <summary>
    /// 依 ProviderId 查出身分來源設定，找不到時回傳 null
    /// </summary>
    public Task<IdentityProvider?> FindAsync(string? providerId)
    {
        return _db.IdentityProviders.AsNoTracking().FirstOrDefaultAsync(p => p.ProviderId == providerId);
    }

    /// <summary>
    /// 依 client_id 查出該 Client 綁定的身分來源：
    /// OpenIddictApplications.Properties.identity_provider_id → IdentityProviders.ProviderId
    /// </summary>
    /// <returns>找不到 client、沒有綁定，或綁定的身分來源不存在時回傳 null</returns>
    public async Task<IdentityProvider?> ResolveByClientIdAsync(string clientId)
    {
        /*
        等同下面SQL：
        SELECT p.*
        FROM "OpenIddictApplications" a
        JOIN "IdentityProviders" p
        ON p."ProviderId" = a."Properties"::jsonb ->> 'identity_provider_id';
        */

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

        return await FindAsync(element.GetString());
    }
}
```

4. OAuth.Core/ServiceCollectionExtensions.cs：註冊兩個 Store
最上面加上 using OAuth.Core.Stores;，在 return services; 前面加上：
``` csharp
// 資料存取：網站的授權流程與背景清除服務使用
services.AddScoped<PendingAuthorizeRequestStore>();
services.AddScoped<IdentityProviderStore>();
```

5. AuthorizationController.cs：整個檔案換成下面這樣
你之前加在檔案裡的中文註解都保留了。和原本相比，只改了「讀寫 DB 的地方改成呼叫 Store」：
``` csharp
using Microsoft.AspNetCore;
using Microsoft.AspNetCore.Authentication;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.WebUtilities;
using OAuth.AuthorizationServer.Services;
using OAuth.Core.Data;
using OAuth.Core.Stores;
using OpenIddict.Abstractions;
using OpenIddict.Server.AspNetCore;
using System.Security.Claims;
using static OpenIddict.Abstractions.OpenIddictConstants;

namespace OAuth.AuthorizationServer.Controllers;

public class AuthorizationController : Controller
{
    private readonly ILogger<AuthorizationController> _logger;
    private readonly PendingAuthorizeRequestStore _pendingStore;
    private readonly IdentityProviderStore _identityProviderStore;
    private readonly IdentityProviderClient _identityProviderClient;
    private readonly IOpenIddictScopeManager _scopeManager;
    private readonly IOpenIddictAuthorizationManager _authorizationManager;
    private readonly IOpenIddictTokenManager _tokenManager;

    public AuthorizationController(
        ILogger<AuthorizationController> logger,
        PendingAuthorizeRequestStore pendingStore,
        IdentityProviderStore identityProviderStore,
        IdentityProviderClient identityProviderClient,
        IOpenIddictScopeManager scopeManager,
        IOpenIddictAuthorizationManager authorizationManager,
        IOpenIddictTokenManager tokenManager)
    {
        _logger = logger;
        _pendingStore = pendingStore;
        _identityProviderStore = identityProviderStore;
        _identityProviderClient = identityProviderClient;
        _scopeManager = scopeManager;
        _authorizationManager = authorizationManager;
        _tokenManager = tokenManager;
    }

    /// <summary>
    /// 處理授權請求，根據是否帶有 __nonce 判斷是第一次還是第二次經過授權流程
    /// </summary>
    /// <param name="resumeNonce">__nonce，用於標識暫存的授權請求，內部使用。 避免撞名</param>
    /// <returns></returns>
    /// <exception cref="InvalidOperationException"></exception>
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

        // 第一次經過：還沒驗證身分，依 client_id 查出綁定的身分來源（設計文件第6節），決定要導去哪裡驗證
        var provider = await _identityProviderStore.ResolveByClientIdAsync(request.ClientId!);
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

        // 暫存原始請求，取得這次流程的 nonce
        var nonce = await _pendingStore.CreateAsync(new PendingAuthorizeData
        {
            ClientId = request.ClientId!,
            RedirectUri = request.RedirectUri!,
            Scope = request.Scope,
            State = request.State,
            CodeChallenge = request.CodeChallenge,
            CodeChallengeMethod = request.CodeChallengeMethod,
        }, provider.ProviderId);

        _logger.LogInformation(
            "Authorize request received. ClientId={ClientId}, Scope={Scope}, redirecting to Bridge.",
            request.ClientId, request.Scope);

        //要從PNP返回的URL
        var callbackUrl = Url.Action(nameof(BridgeCallback), "Authorization", values: null, protocol: Request.Scheme)!;
        //組成要導向PNP的Bridge URL
        var bridgeUrl = $"{provider.BridgeUrl}?return_url={Uri.EscapeDataString(callbackUrl)}&nonce={Uri.EscapeDataString(nonce)}";

        return Redirect(bridgeUrl);
    }

    /// <summary>
    /// PNP Callback。並把PNP驗證資料存入 PendingAuthorizeRequest
    /// </summary>
    /// <param name="bridgeCode">PNP產生，確認通過PNP登入驗證</param>
    /// <param name="nonce">Authorize產生</param>
    /// <returns></returns>
    [HttpGet("~/connect/authorize/bridge-callback")]
    public async Task<IActionResult> BridgeCallback([FromQuery] string? bridgeCode, [FromQuery] string? nonce)
    {
        // 輸入驗證（比照SECURITY.md）：必填、長度上限、null byte防護
        if (!IsValidCallbackParameter(bridgeCode) || !IsValidCallbackParameter(nonce))
        {
            _logger.LogWarning("BridgeCallback rejected. Reason=InvalidParameter.");
            return BadRequest("缺少或格式錯誤的參數。");
        }

        // 先確認nonce有效、而且還沒回填過身分，才去呼叫Redeem，避免浪費PNP的一次性code
        var pending = await _pendingStore.FindFillableAsync(nonce!);
        if (pending is null)
        {
            _logger.LogWarning("BridgeCallback rejected. Reason=InvalidOrExpiredNonce.");
            return BadRequest("無效或已過期的 nonce。");
        }

        // 依第一次經過時記下的身分來源，決定要去哪裡Redeem
        var provider = await _identityProviderStore.FindAsync(pending.ProviderId);
        if (provider is null)
        {
            _logger.LogError("BridgeCallback rejected. ProviderId={ProviderId}, Reason=IdentityProviderNotFound.", pending.ProviderId);
            return BadRequest("身分來源設定不存在。");
        }

        //用bridgeCode換回PNP的身分資料{userName, siteCode, tokenId, ticketExpiration}
        var redeem = await _identityProviderClient.RedeemAsync(provider, bridgeCode!, nonce!);
        if (redeem is null)
        {
            _logger.LogWarning("BridgeCallback rejected. ProviderId={ProviderId}, Reason=RedeemFailed.", provider.ProviderId);
            return BadRequest("身分驗證失敗，請重新登入。");
        }
        var (userName, siteCode, tokenId, ticketExpirationUnix) = redeem;

        // 原子性地回填身分：同一個nonce收到多次bridge-callback時，只有第一次能寫入
        var filled = await _pendingStore.TryFillIdentityAsync(nonce!, userName, siteCode, tokenId,
            DateTimeOffset.FromUnixTimeSeconds(ticketExpirationUnix));
        if (!filled)
        {
            _logger.LogWarning("BridgeCallback rejected. Reason=AlreadyFilledOrExpired.");
            return BadRequest("無效或已過期的 nonce。");
        }

        _logger.LogInformation(
            "BridgeCallback succeeded. UserName={UserName}, SiteCode={SiteCode}, TokenId={TokenId}.",
            userName, siteCode, tokenId);

        // 繞回 /connect/authorize 本身（設計文件第4.1節說明框），帶上重建的原始參數與 __nonce
        var original = PendingAuthorizeRequestStore.ReadOriginalRequest(pending);

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
            queryParams.Where(p => p.Value is not null).ToDictionary(p => p.Key, p => p.Value));

        return Redirect(authorizeUrl);
    }

    /// <summary>
    /// 交換授權碼或其他 grant type 以取得 access_token。
    /// </summary>
    /// <returns></returns>
    /// <exception cref="InvalidOperationException"></exception>
    /// <exception cref="NotImplementedException"></exception>
    [HttpPost("~/connect/token")]
    public async Task<IActionResult> Exchange()
    {
        var request = HttpContext.GetOpenIddictServerRequest()
            ?? throw new InvalidOperationException("OpenIddict 請求物件遺失。");

        if (request.IsAuthorizationCodeGrantType())
        {
            // OpenIddict 在這之前已經自動驗證過：
            // 1. Authorization Code 本身有效、未過期、未使用過
            // 2. PKCE：SHA256(code_verifier) == 當初的 code_challenge
            // 我們只要把 CompleteAuthorizationAsync() 當初存進這個Authorization Code裡的
            // ClaimsPrincipal 拿回來，再呼叫一次 SignIn()，OpenIddict 就會依此簽發 access_token/refresh_token。
            var result = await HttpContext.AuthenticateAsync(OpenIddictServerAspNetCoreDefaults.AuthenticationScheme);

            if (result.Principal is null)
            {
                _logger.LogWarning("Token exchange rejected. ClientId={ClientId}, GrantType=authorization_code, Reason=MissingPrincipal.", request.ClientId);
                return BadRequest("無法取得授權時建立的身分資料。");
            }

            _logger.LogInformation(
                "AccessToken issued. ClientId={ClientId}, UserName={UserName}, Scope={Scope}, GrantType=authorization_code.",
                request.ClientId, result.Principal.GetClaim(Claims.Subject), request.Scope);

            return SignIn(result.Principal, OpenIddictServerAspNetCoreDefaults.AuthenticationScheme);
        }
        else if (request.IsRefreshTokenGrantType())
        {
            var result = await HttpContext.AuthenticateAsync(OpenIddictServerAspNetCoreDefaults.AuthenticationScheme);

            if (result.Principal is null)
            {
                return RejectRefreshToken("無法取得 refresh token 建立時的身分資料。", null);
            }

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
            var provider = await _identityProviderStore.FindAsync(providerId);
            if (provider is null)
            {
                await RevokeTokenChainAsync(result.Principal);
                return RejectRefreshToken("身分來源設定不存在。", userName);
            }

            // 第1層：向身分來源查詢 tokenId 是否已被登出/被踢（查詢失敗也視為未通過）
            var isActive = await _identityProviderClient.IsTokenActiveAsync(provider, int.Parse(tokenIdClaim));
            if (!isActive)
            {
                await RevokeTokenChainAsync(result.Principal);
                return RejectRefreshToken("PNP tokenId 已被登出/被踢。", userName);
            }

            // 第2層：ticketExpiration 絕對上限
            var ticketExpiration = DateTimeOffset.FromUnixTimeSeconds(long.Parse(ticketExpirationClaim));
            if (DateTimeOffset.UtcNow >= ticketExpiration)
            {
                await RevokeTokenChainAsync(result.Principal);
                return RejectRefreshToken("已超過 ticketExpiration 絕對上限，需重新走 Bridge。", userName);
            }

            _logger.LogInformation(
                "AccessToken issued. ClientId={ClientId}, UserName={UserName}, Scope={Scope}, GrantType=refresh_token.",
                request.ClientId, userName, request.Scope);

            return SignIn(result.Principal, OpenIddictServerAspNetCoreDefaults.AuthenticationScheme);
        }

        throw new NotImplementedException("此 grant type 尚未實作。");
    }

    /// <summary>
    /// 完成授權流程，並簽發 access_token
    /// </summary>
    /// <param name="resumeNonce"></param>
    /// <param name="request"></param>
    /// <returns></returns>
    private async Task<IActionResult> CompleteAuthorizationAsync(string resumeNonce, OpenIddictRequest request)
    {
        // 原子性地「檢查有效 + 標記已使用」：同一個nonce同時被送多次時，只有一個請求能取得
        var pending = await _pendingStore.TryConsumeAsync(resumeNonce);
        if (pending is null)
        {
            _logger.LogWarning("Authorization rejected. ClientId={ClientId}, Reason=InvalidOrExpiredPendingRequest.", request.ClientId);
            return BadRequest("無效的授權狀態，請重新開始登入流程。");
        }

        // 第二次經過帶來的參數，必須跟第一次經過時暫存的原始參數完全一致，
        // 避免resume網址在瀏覽器端被竄改（例如換掉code_challenge、redirect_uri、scope）
        var original = PendingAuthorizeRequestStore.ReadOriginalRequest(pending);
        if (!MatchesOriginalRequest(request, original))
        {
            // 參數被竄改視為異常事件：TryConsumeAsync已經標記Used=true，這筆pending已作廢，不給重試機會
            _logger.LogWarning("Authorization rejected. ClientId={ClientId}, Reason=ResumeParametersTampered.", request.ClientId);
            return BadRequest("授權參數與原始請求不一致，請重新開始登入流程。");
        }

        var identity = new ClaimsIdentity(
            authenticationType: "PNP-Bridge",
            nameType: Claims.Name,
            roleType: Claims.Role);

        identity.SetClaim(Claims.Subject, pending.UserName);
        identity.SetClaim("idp", pending.ProviderId);
        identity.SetClaim("site_code", pending.SiteCode);
        identity.SetClaim("token_id", pending.TokenId?.ToString());
        identity.SetClaim("ticket_expiration", pending.TicketExpiration?.ToUnixTimeSeconds().ToString());
        identity.SetScopes(request.GetScopes());

        // 依本次授權的 scope，查出它們對應的 Resource Server（OpenIddictScopes.Resources 欄位），
        identity.SetScopes(request.GetScopes());

        // 依本次授權的 scope，查出它們對應的 Resource Server（OpenIddictScopes.Resources 欄位），
        // OpenIddict 簽發 JWT 時會把這些值轉成 aud claim
        identity.SetResources(await _scopeManager.ListResourcesAsync(identity.GetScopes()).ToListAsync());

        var principal = new ClaimsPrincipal(identity);

        foreach (var claim in principal.Claims)
        {
            //claim.SetDestinations(Destinations.AccessToken)做的事，就是把一個標記寫進這個claim物件自己的Properties字典裡，內容大概等於：「這個claim，將來要被放進access_token裡」。
            //程式碼裡對site_code、token_id、ticket_expiration這幾個自訂claim都呼叫SetDestinations(Destinations.AccessToken)，就是在明確表態：「這幾個claim，請放進access_token」，之後你把拿到的JWT丟去jwt.io解開，應該就會在payload裡看到這些欄位。
            claim.SetDestinations(Destinations.AccessToken);
        }

        _logger.LogInformation(
            "AuthorizationCode issued. ClientId={ClientId}, UserName={UserName}, Scope={Scope}.",
            request.ClientId, pending.UserName, request.Scope);

        return SignIn(principal, OpenIddictServerAspNetCoreDefaults.AuthenticationScheme);
    }

    /// <summary>
    /// 拒絕 Refresh Token 的請求，並記錄原因與使用者名稱
    /// </summary>
    /// <param name="reason"></param>
    /// <param name="userName"></param>
    /// <returns></returns>
    private IActionResult RejectRefreshToken(string reason, string? userName)
    {
        _logger.LogWarning("RefreshToken rejected. UserName={UserName}, Reason={Reason}.", userName, reason);

        var properties = new AuthenticationProperties(new Dictionary<string, string?>
        {
            [OpenIddictServerAspNetCoreConstants.Properties.Error] = Errors.InvalidGrant,
            [OpenIddictServerAspNetCoreConstants.Properties.ErrorDescription] = reason,
        });

        return Forbid(properties, OpenIddictServerAspNetCoreDefaults.AuthenticationScheme);
    }

    /// <summary>
    /// 撤銷指定使用者的所有相關 Token 鏈（包括 Access Token 和 Refresh Token）
    /// </summary>
    /// <param name="principal"></param>
    /// <returns></returns>
    private async Task RevokeTokenChainAsync(ClaimsPrincipal principal)
    {
        var authorizationId = principal.GetAuthorizationId();
        if (string.IsNullOrEmpty(authorizationId))
        {
            _logger.LogWarning("RevokeTokenChain skipped. UserName={UserName}, Reason=MissingAuthorizationId.",
                principal.GetClaim(Claims.Subject));
            return;
        }

        var authorization = await _authorizationManager.FindByIdAsync(authorizationId);
        if (authorization is not null)
        {
            await _authorizationManager.TryRevokeAsync(authorization);
        }

        var revokedCount = await _tokenManager.RevokeByAuthorizationIdAsync(authorizationId);

        _logger.LogWarning(
            "RevokeTokenChain completed. UserName={UserName}, AuthorizationId={AuthorizationId}, RevokedTokens={RevokedCount}.",
            principal.GetClaim(Claims.Subject), authorizationId, revokedCount);
    }

    /// <summary>
    /// 比對第二次經過/connect/authorize的參數，是否與第一次經過時暫存的原始參數一致
    /// </summary>
    private static bool MatchesOriginalRequest(OpenIddictRequest request, PendingAuthorizeData original)
    {
        return string.Equals(request.ClientId, original.ClientId, StringComparison.Ordinal)
            && string.Equals(request.RedirectUri, original.RedirectUri, StringComparison.Ordinal)
            && string.Equals(request.Scope, original.Scope, StringComparison.Ordinal)
            && string.Equals(request.State, original.State, StringComparison.Ordinal)
            && string.Equals(request.CodeChallenge, original.CodeChallenge, StringComparison.Ordinal)
            && string.Equals(request.CodeChallengeMethod, original.CodeChallengeMethod, StringComparison.Ordinal);
    }

    /// <summary>
    /// bridge-callback參數的基本格式檢查：必填、長度上限、不可含null byte
    /// </summary>
    private static bool IsValidCallbackParameter(string? value)
    {
        return !string.IsNullOrEmpty(value)
            && value.Length <= 256
            && !value.Contains('\0');
    }
}
```

6. PendingAuthorizeRequestCleanupService.cs：改用 Store
最上面的 using 改成：
``` csharp
using OAuth.Core.Stores;
```
（原本的 using Microsoft.EntityFrameworkCore;、using OAuth.Core.Data; 可以刪掉。）

try 裡面改成：
``` csharp
// Store 是 Scoped，背景服務是 Singleton，不能直接注入，每次清除時自己開一個 scope
using var scope = _scopeFactory.CreateScope();
var store = scope.ServiceProvider.GetRequiredService<PendingAuthorizeRequestStore>();

var deleted = await store.DeleteExpiredAsync(stoppingToken);

_logger.LogInformation("PendingAuthorizeRequest cleanup completed. Deleted={Deleted}.", deleted);
```

7. 新增 OAuth.Tests/PendingAuthorizeRequestStoreTests.cs
直接測 Store，不經過 HTTP：
``` csharp
using Microsoft.Extensions.DependencyInjection;
using OAuth.Core.Data;
using OAuth.Core.Stores;
using OAuth.Tests.Infrastructure;

namespace OAuth.Tests;

/// <summary>
/// 直接測試 PendingAuthorizeRequestStore 的 DB 邏輯（不經過 HTTP 與 OAuth 流程）
/// </summary>
[Collection(AuthServerCollection.Name)]
public class PendingAuthorizeRequestStoreTests
{
    private readonly AuthServerFactory _factory;

    public PendingAuthorizeRequestStoreTests(AuthServerFactory factory)
    {
        _factory = factory;
    }

    private static PendingAuthorizeData Original() => new()
    {
        ClientId = OAuthTestClient.ClientId,
        RedirectUri = OAuthTestClient.RedirectUri,
        Scope = OAuthTestClient.Scope,
    };

        [Fact]
    public async Task 回填身分_同一個nonce只有第一次成功()
    {
        using var scope = _factory.Services.CreateScope();
        var store = scope.ServiceProvider.GetRequiredService<PendingAuthorizeRequestStore>();
        var nonce = await store.CreateAsync(Original(), "pnp");

        var first = await store.TryFillIdentityAsync(nonce, "user-a", "AS", 1, DateTimeOffset.UtcNow.AddHours(1));
        var second = await store.TryFillIdentityAsync(nonce, "user-b", "AS", 2, DateTimeOffset.UtcNow.AddHours(1));

        Assert.True(first);
        Assert.False(second);
    }

    [Fact]
    public async Task 取得使用權_尚未回填身分時失敗()
    {
        using var scope = _factory.Services.CreateScope();
        var store = scope.ServiceProvider.GetRequiredService<PendingAuthorizeRequestStore>();
        var nonce = await store.CreateAsync(Original(), "pnp");

        Assert.Null(await store.TryConsumeAsync(nonce));
    }    

    [Fact]
    public async Task 取得使用權_同一個nonce只有第一次成功()
    {
        using var scope = _factory.Services.CreateScope();
        var store = scope.ServiceProvider.GetRequiredService<PendingAuthorizeRequestStore>();
        var nonce = await store.CreateAsync(Original(), "pnp");
        await store.TryFillIdentityAsync(nonce, "user-a", "AS", 1, DateTimeOffset.UtcNow.AddHours(1));

        var first = await store.TryConsumeAsync(nonce);
        var second = await store.TryConsumeAsync(nonce);

        Assert.NotNull(first);
        Assert.Equal("user-a", first.UserName);
        Assert.True(first.Used);
        Assert.Null(second);
    }    

    [Fact]
    public async Task 讀出原始請求_與建立時相同()
    {
        using var scope = _factory.Services.CreateScope();
        var store = scope.ServiceProvider.GetRequiredService<PendingAuthorizeRequestStore>();
        var nonce = await store.CreateAsync(Original(), "pnp");
        await store.TryFillIdentityAsync(nonce, "user-a", "AS", 1, DateTimeOffset.UtcNow.AddHours(1));

        var pending = await store.TryConsumeAsync(nonce);
        var original = PendingAuthorizeRequestStore.ReadOriginalRequest(pending!);

        Assert.Equal(OAuthTestClient.ClientId, original.ClientId);
        Assert.Equal(OAuthTestClient.RedirectUri, original.RedirectUri);
        Assert.Equal(OAuthTestClient.Scope, original.Scope);
    }
}
```

幾個重點說明：

- Store 是什麼：一個專門負責「某一類資料怎麼讀寫」的類別。Controller 只說「幫我回填身分」，不需要知道背後是 ExecuteUpdateAsync、WHERE 條件怎麼寫。之後要調整 DB 邏輯，例如加索引、改條件，只要改 Store。
- 方法名稱用 Try 開頭：TryFillIdentityAsync、TryConsumeAsync 表示「可能不成功，而且不成功是正常情況」，用回傳值（false、null）告訴呼叫者，不丟例外。這是 .NET 常見的命名慣例，例如 int.TryParse。
- nonce 的產生也移到 Store：nonce 就是 pending 的主鍵，「產生主鍵」屬於建立資料的一部分，所以和 CreateAsync 放在一起。
- ReadOriginalRequest 是 static：它只是把 JSON 轉回物件，不需要 DB，Controller 在兩個地方都要用。
- Controller 的建構子少了 AuthorizationServerDbContext 和 IOpenIddictApplicationManager：這兩個現在只有 Store 會用到。Controller 不再直接碰 DB，這也是這次重構的目標。
- 為什麼 Store 測試只需要 Services.CreateScope()：測試直接從 DI 容器拿 Store 來呼叫，不用組 HTTP 請求、不用跑 OAuth 流程。這就是抽出來的好處：DB 邏輯可以單獨測試，測試更短，失敗時也更容易知道是哪裡出錯。

測試時要注意：Authorization Server 要先停掉，避免 dll 被鎖住。

1. 編譯並執行全部測試：
dotnet build OAuth.slnx
dotnet test OAuth.Tests
2. 啟動 Authorization Server 和 MockPnp，手動跑一次完整流程，確認實際執行也正常。

請執行後確認：
1. 編譯沒有錯誤，也不再有 AddQueryString 的可 null 警告。
2. dotnet test 顯示 通過: 34，失敗: 0：原本 30 個全部通過，代表重構沒有改變任何行為；再加上新的 4 個 Store 測試。
3. 手動的完整流程正常。