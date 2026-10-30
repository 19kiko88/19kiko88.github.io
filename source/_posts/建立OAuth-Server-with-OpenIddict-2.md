---
title: 建立OAuth Server with OpenIddict(2)
date: 2026-10-27 17:44:03
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
## 資安加固 ：限定aud（audience）claim

這一步在做什麼：確保JWT的aud（受眾/目標對象）明確寫上「這個token只給某個特定的Resource Server用」，而不是一個沒有限定範圍的通用token。OpenIddict的做法是：把「資源（resource）」關聯在scope上，當某個scope被授予時，OpenIddict會自動把該scope關聯的resource加進aud裡，不需要我們手動在Controller程式碼裡額外處理。

- **修改Step 7的seed程式碼，幫每個scope加上Resources**

在Program.cs的SeedDataAsync()裡，把scope的定義從：
``` csharp
var scopeDescriptors = new[]
{
    new OpenIddictScopeDescriptor { Name = "ucs.sn-info.read", DisplayName = "SN 資料讀取" },
    new OpenIddictScopeDescriptor { Name = "ucs.ecn-rcn.read", DisplayName = "UCS ECN/RCN 資料讀取" },
    new OpenIddictScopeDescriptor { Name = "pnp.auto-open-pnp.read", DisplayName = "SN自動開啟點圖 讀取" },
};
```
改成每個都加Resources：
``` csharp
const string mcpResourceServerId = "urn:pnp:ucs-mcp"; // TODO：等真正的MCP Resource Server確定網址/識別碼後再更新

var scopeDescriptors = new[]
{
    new OpenIddictScopeDescriptor { Name = "ucs.sn-info.read", DisplayName = "SN 資料讀取", Resources = { mcpResourceServerId } },
    new OpenIddictScopeDescriptor { Name = "ucs.ecn-rcn.read", DisplayName = "UCS ECN/RCN 資料讀取", Resources = { mcpResourceServerId } },
    new OpenIddictScopeDescriptor { Name = "pnp.auto-open-pnp.read", DisplayName = "SN自動開啟點圖 讀取", Resources = { mcpResourceServerId } },
};
```

說明：mcpResourceServerId目前先用一個佔位的URN字串（urn:pnp:ucs-mcp），代表「包裝UcsService的那個MCP Resource Server」。之後MCP Resource Server真的做出來、有實際網址後，要回來更新這個值，並且兩邊要一致——MCP Resource Server驗證JWT時，會檢查aud是不是等於這個值，兩邊對不上就會拒絕。

- **AuthorizationController.cs：注入 IOpenIddictScopeManager**
``` csharp
private readonly AuthorizationServerDbContext _db;
private readonly IConfiguration _configuration;
private readonly ILogger<AuthorizationController> _logger;
private readonly IOpenIddictScopeManager _scopeManager;

public AuthorizationController(
    AuthorizationServerDbContext db,
    IConfiguration configuration,
    ILogger<AuthorizationController> logger,
    IOpenIddictScopeManager scopeManager)
{
    _db = db;
    _configuration = configuration;
    _logger = logger;
    _scopeManager = scopeManager;
}
```

CompleteAuthorizationAsync()：在 identity.SetScopes(...) 下面加一行
``` csharp
// 依本次授權的 scope，查出它們對應的 Resource Server（OpenIddictScopes.Resources 欄位），
// OpenIddict 簽發 JWT 時會把這些值轉成 aud claim
identity.SetResources(await _scopeManager.ListResourcesAsync(identity.GetScopes()).ToListAsync());
```

- **重建scope資料讓新設定生效**
因為scope的seed邏輯是「查無這個名稱的scope才建立」，既有的3筆scope資料不會因為程式碼改了就自動更新，一樣要先手動刪除：
``` sql
DELETE FROM "OpenIddictScopes" WHERE "Name" IN ('ucs.sn-info.read', 'ucs.ecn-rcn.read', 'pnp.auto-open-pnp.read');
```

刪完重新F5跑起來，<span style="color: red;">讓seed重新建立這3筆（這次會帶著Resources）。</span>

請執行後確認：
1. dotnet build正常編譯。
2. 重新完整走一次流程拿新的access_token，丟去jwt.io解開，確認payload裡出現aud欄位，值是urn:pnp:ucs-mcp。
#
#
#
---
## refresh 被拒絕時，撤銷整條 refresh token 鏈（設計文件第 4.3 節）

這一步在做什麼：你寫好的雙層檢查不通過時，目前只會拒絕「這一次」的 refresh 請求，同一條鏈上其他還有效的 token 都不會被作廢，也就是 Exchange() 裡那兩個 // TODO：撤銷整條refresh token鏈。這次把它補上。

先說明 OpenIddict 怎麼串起「一條鏈」：使用者走完一次 /connect/authorize 時，OpenIddict 會自動在 OpenIddictAuthorizations 表建一筆授權紀錄。我們的 client 是 Implicit consent，這筆紀錄是 ad-hoc 類型。之後由這次登入衍生的 Authorization Code、每一代 refresh_token、access_token，在 OpenIddictTokens 表裡的 AuthorizationId 都指向同一筆授權。所以「撤銷整條鏈」就是撤銷這筆授權，加上它底下所有的 token。

- **AuthorizationController.cs：注入兩個 manager (IOpenIddictAuthorizationManager. IOpenIddictTokenManager)**
``` csharp
private readonly IOpenIddictScopeManager _scopeManager;
private readonly IOpenIddictAuthorizationManager _authorizationManager;
private readonly IOpenIddictTokenManager _tokenManager;
private readonly IOpenIddictAuthorizationManager _authorizationManager;
private readonly IOpenIddictTokenManager _tokenManager;

public AuthorizationController(
    AuthorizationServerDbContext db,
    IConfiguration configuration,
    ILogger<AuthorizationController> logger,
    IOpenIddictScopeManager scopeManager,
    IOpenIddictAuthorizationManager authorizationManager,
    IOpenIddictTokenManager tokenManager)
{
    _db = db;
    _configuration = configuration;
    _logger = logger;
    _scopeManager = scopeManager;
    _authorizationManager = authorizationManager;
    _tokenManager = tokenManager;
}
```

- **在AuthorizationController 新增一個 private 方法**
``` csharp
/// <summary>
/// 撤銷整條refresh token鏈：撤銷這次登入對應的Authorization，以及其底下所有token
/// </summary>
/// <param name="principal">refresh token 解出來的身分資料</param>
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
        "RevokeTokenChain completed. UserName={UserName}, Auth, RevokedTokens={RevokedCount}.",
        principal.GetClaim(Claims.Subject), authorizationId, revokedCount);
}
```

- **Exchange() 的 refresh_token 分支：principal 存在但檢查失敗的三個地方，加上撤銷Token的實際Function再拒絕**
``` csharp
if (tokenIdClaim is null || ticketExpirationClaim is null)
{
    await RevokeTokenChainAsync(result.Principal);
    return RejectRefreshToken("缺少撤銷檢查所需的 claim。", userName);
}

// 第1層：IsTokenActive（目前mock，永遠回傳true；之後改成真的呼叫PCBRepairNavi）
var isActive = await IsTokenActiveAsync(int.Parse(tokenIdClaim));
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
```

幾個重點說明：
- principal.GetAuthorizationId()：OpenIddict 會把授權 ID 存在 principal 的內部 claim（oi_au_id）裡，refresh token 解出來的 principal 也有，所以可以直接拿到。
- TryRevokeAsync(authorization)：把 OpenIddictAuthorizations.Status 改成 revoked。OpenIddict 之後驗證任何 token 時都會檢查它所屬的授權是不是還有效，所以單做這一步，整條鏈其實就已經不能用了。
- RevokeByAuthorizationIdAsync(authorizationId)：把底下每一筆 token 的 Status 也改成 revoked。這是為了 DB 狀態明確、稽核時一眼看得出來，不用再回頭對照授權表。
- 為什麼 result.Principal is null 那個分支不撤銷：那種情況連 principal 都拿不到，也就拿不到 AuthorizationId，無從撤銷。實際上 OpenIddict 對無效的 refresh token 會在進到我們的 action 之前就先擋掉，這個分支只是防禦用。

測試時要注意：mock 的 IsTokenActiveAsync 永遠回傳 true，所以這次用第 2 層（ticketExpiration）來觸發拒絕。

1. 暫時把 mock 的 RedeemAsync 回傳值從 DateTimeOffset.UtcNow.AddHours(12) 改成 DateTimeOffset.UtcNow.AddMinutes(1)，然後重啟 Server。
2. 跑一次完整流程拿到 access_token 和 refresh_token，scope 要含 offline_access。
3. 馬上打一次 refresh，預期 200，這時還沒過期。把新拿到的 refresh_token 記下來。
4. 等 1 分鐘以上，再用新的 refresh_token 打一次 refresh。
5. 測完記得把 RedeemAsync 改回 AddHours(12)。

{% asset_image ticket_expired.jpg ticket_expired%}
#
#
#
---
## 第二次經過 /connect/authorize 時，檢查參數有沒有被竄改

這一步在做什麼：bridge-callback 會把原始參數和 __nonce 組成網址，讓瀏覽器導回 /connect/authorize。問題是這個網址在瀏覽器手上，任何人都能改，而 CompleteAuthorizationAsync() 目前只查 __nonce 是否有效，完全沒有比對這次帶來的參數，是不是和第一次暫存在 PendingAuthorizeRequest.OriginalRequestJson 裡的一樣。

可能的攻擊情境：攻擊者拿到受害者的 resume 網址，例如從 log、Referer 或瀏覽器歷史紀錄。他把裡面的 code_challenge 換成自己產生的值，搶在受害者之前送出。OpenIddict 就會以受害者的身分簽出 Authorization Code，而且這個 code 和攻擊者的 code_challenge 綁在一起，攻擊者能用自己的 code_verifier 換到受害者的 token。<span style="color: red;">PKCE 原本要擋的就是這種事，但因為我們多繞了一手，這道防線在中間被繞過了。</span>

- **CompleteAuthorizationAsync()：在檢查 pending 有效之後、pending.Used = true 之前，加上比對**
``` csharp
if (pending is null || pending.Used || pending.ExpiresAt < DateTimeOffset.UtcNow || pending.UserName is null)
{
    _logger.LogWarning("Authorization rejected. ClientId={ClientId}, Reason=InvalidOrExpiredPendingRequest.", request.ClientId);
    return BadRequest("無效的授權狀態，請重新開始登入流程。");
}

// 第二次經過帶來的參數，必須跟第一次經過時暫存的原始參數完全一致，
// 避免resume網址在瀏覽器端被竄改（例如換掉code_challenge、redirect_uri、scope）
var original = JsonSerializer.Deserialize<PendingAuthorizeData>(pending.OriginalRequestJson)!;
if (!MatchesOriginalRequest(request, original))
{
    // 參數被竄改視為異常事件：直接作廢這筆pending，不給重試機會
    pending.Used = true;
    await _db.SaveChangesAsync();

    _logger.LogWarning("Authorization rejected. ClientId={ClientId}, Reason=ResumeParametersTampered.", request.ClientId);
    return BadRequest("授權參數與原始請求不一致，請重新開始登入流程。");
}

pending.Used = true;
await _db.SaveChangesAsync();
```

- **新增一個 private 方法**
``` csharp
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
```

幾個重點說明：

- 為什麼 6 個欄位全部都要比對：
  - code_challenge 和 code_challenge_method：防止上面那種 PKCE 替換攻擊。
  - redirect_uri：防止 code 被導去別的地方。OpenIddict 雖然會檢查網址有沒有註冊，但同一個 client 有多個 redirect_uri 時還是可以互換。
  - scope：防止有人偷偷把權限擴大。
  - client_id 和 state：確保整個請求和原始請求是同一個。
- 為什麼用 StringComparison.Ordinal：這些都是協定值，必須逐字元完全相符。不能忽略大小寫，也不能受文化特性影響。
- 為什麼不一致時要把 pending 作廢：參數被改過，代表這個 resume 網址可能已經外洩。作廢之後，就算攻擊者手上有原始網址也不能再用，受害者最多只是要重新登入一次。
- OriginalRequestJson 為什麼可以信任：它存在 Authorization Server 自己的 DB，瀏覽器碰不到。BridgeCallback() 也是從這裡重建參數，所以正常流程下兩邊一定會一致。

測試時要注意：要竄改的是 bridge-callback 回應的 302 Location，也就是那個帶 __nonce 的 /connect/authorize?... 網址。

測試流程：
1. 先關閉Follow redirects(Insomnia的設定：Application => Preferences => Follow redirects，把他取消勾選)
2. 正常流程呼叫，api [connect/authorize]. [connect/authorize/bridge-callback]
3. 這步驟是種點，要模擬<span>PNP角色</span>呼叫OAuth Server的API [connect/authorize]。參數要與API [connect/authorize]的Header的location一致。
{% asset_image mock_api_params_setting.jpg mock_api_params_setting %}
4. 接著把code_challenge改動一個字元(長度不變)，讓他跟原始的code_challenge不一致。正常的話會出現[授權參數與原始請求不一致，請重新開始登入流程]的錯誤訊息。
{% asset_image mock_api.jpg mock_api %}
5. 到DB確認 那筆 PendingAuthorizeRequests 的 Used 變成 true。
6. 再送一次沒改過的原始網址，預期回 400「無效的授權狀態」。證明 pending 已經作廢，攻擊者也不能拿原始網址再用。
{% asset_image pending_revoke.jpg pending_revoke %}

請執行後確認：
1. dotnet build 正常編譯。
2. 到DB確認 那筆 PendingAuthorizeRequests 的 Used 變成 true。
3. 正常流程一樣能正常拿到 code，換到 token。
4. <span style="color: red;">Follow redirects開啟</span>
#
#
#
---
## 讓 PendingAuthorizeRequest「單次使用」在同時有多個請求時也成立

這一步在做什麼：CompleteAuthorizationAsync() 目前的寫法是「先讀出來 → 檢查 Used == false → 設 Used = true → 存檔」，分成兩次 DB 操作。如果同一個 resume 網址在極短時間內被送了兩次，例如攻擊者和受害者同時送出，可能會發生：

請求A：讀到 Used=false ─┐
請求B：讀到 Used=false ─┤  兩個都通過檢查
請求A：寫入 Used=true   │
請求B：寫入 Used=true  ─┘  → 兩個都簽出 Authorization Code

這叫 race condition，「單次使用」被打破。解法是把「檢查」和「標記」合成一條 SQL，由資料庫保證只有一個請求能成功：
``` sql
UPDATE "PendingAuthorizeRequests" SET "Used" = true
WHERE "Nonce" = @nonce AND "Used" = false AND "ExpiresAt" > @now AND "UserName" IS NOT NULL
```

PostgreSQL 執行 UPDATE 時會鎖住那一列，第二個請求要等第一個做完才能檢查，那時 Used 已經是 true，條件不成立，影響筆數是 0。看影響筆數是 1 還是 0，就知道自己有沒有搶到。
CompleteAuthorizationAsync()：把開頭到 pending.Used = true; await _db.SaveChangesAsync(); 這一段整個換掉
``` csharp
private async Task<IActionResult> CompleteAuthorizationAsync(string resumeNonce, OpenIddictRequest request)
{
    // 用單一條UPDATE原子性地「檢查有效 + 標記已使用」：
    // 同一個nonce同時被送多次時，只有一個請求能讓影響筆數=1，其他都是0
    var now = DateTimeOffset.UtcNow;
    var consumed = await _db.PendingAuthorizeRequests
        .Where(p => p.Nonce == resumeNonce && !p.Used && p.ExpiresAt > now && p.UserName != null)
        .ExecuteUpdateAsync(s => s.SetProperty(p => p.Used, true));

    if (consumed == 0)
    {
        _logger.LogWarning("Authorization rejected. ClientId={ClientId}, Reason=InvalidOrExpiredPendingRequest.", request.ClientId);
        return BadRequest("無效的授權狀態，請重新開始登入流程。");
    }

    // 搶到使用權後，才把資料讀出來（此時Used已經是true）
    var pending = await _db.PendingAuthorizeRequests.AsNoTracking().FirstAsync(p => p.Nonce == resumeNonce);

    // 第二次經過帶來的參數，必須跟第一次經過時暫存的原始參數完全一致，
    // 避免resume網址在瀏覽器端被竄改（例如換掉code_challenge、redirect_uri、scope）
    var original = JsonSerializer.Deserialize<PendingAuthorizeData>(pending.OriginalRequestJson)!;
    if (!MatchesOriginalRequest(request, original))
    {
        // 參數被竄改視為異常事件：上面已經標記Used=true，這筆pending已作廢，不給重試機會
        _logger.LogWarning("Authorization rejected. ClientId={ClientId}, Reason=ResumeParametersTampered.", request.ClientId);
        return BadRequest("授權參數與原始請求不一致，請重新開始登入流程。");
    }

    var identity = new ClaimsIdentity(
    // ...（以下維持不變）
```

幾個重點說明：

- ExecuteUpdateAsync：EF Core 提供的批次更新。它不會先把資料讀進記憶體，而是直接轉成一條 UPDATE ... WHERE ... 送到 DB，回傳影響筆數。原本「讀 → 改 → SaveChangesAsync()」那套寫法，中間就是 race 的空隙。
- 順序反過來了：先標記，再比對參數。原本是「比對 → 不一致就作廢 → 一致也作廢 → 繼續」，不管結果如何最後 Used 都是 true。所以直接一開始就原子性地標記，竄改那條路徑也不用再自己寫 pending.Used = true 了，程式反而更短。
- AsNoTracking()：這筆資料只用來讀，不會再改，所以不需要 EF 追蹤變更。
- FirstAsync 而不是 FirstOrDefaultAsync：前面 UPDATE 影響了 1 筆，就代表這筆一定存在。萬一真的讀不到，那是預期之外的錯誤，讓它丟例外比較合理。
- 拒絕的訊息和 log 都沒變，上一步的竄改測試和作廢測試結果應該維持一樣。

測試時要注意：race 用手動點兩下是很難重現的，要用指令同時送出多個請求。

1. 跑完整流程到 bridge-callback，Follow redirects 關掉，複製 302 的 Location，前面補上 https://localhost:7280。這次不要竄改，也先不要送出。
2. 在 PowerShell 7 用下面的指令同時送 5 次，記得把網址換成你的：
``` bash
 $url = 'https://localhost:7280/connect/authorize?client_id=pnp-mcp-client&redirect_uri=https%3A%2F%2Flocalhost%3A5055%2Fcallback&response_type=code&scope=pnp.L1-L2-previous-rma-info.read%20ucs.sn-info.read%20offline_access&code_challenge=8cufwPs6brroNfGpDSo_lPESHMRg9ZekH-HadwOlZ9k&code_challenge_method=S256&__nonce=GFisanz2A_imuwj-iLS5RKrwGNf_dDxoQ7Ia__DJk6Y'

 curl.exe -k -s -Z --parallel-immediate -w "%{http_code}\n" -o NUL $url -o NUL $url -o NUL $url -o NUL $url -o NUL $url
```
   網址要用單引號包住，因為裡面有 &、$ 這類 PowerShell 會解讀的字元。
3. 再從頭跑一次完整流程，什麼都不改，確認正常流程沒被改壞，可以換到 token。

請執行後確認：
1. dotnet build 正常編譯。
2. 第 2 步的 5 個結果裡，剛好一個 302，其他四個都是 400。
3. Console 有 1 行 AuthorizationCode issued，4 行 Reason=InvalidOrExpiredPendingRequest。
4. 第 3 步的正常流程可以換到 token。

{% asset_image race_condition_test.jpg race_condition_test %}
結果符合預期：同一個 __nonce 同時送 5 次，只有一個請求能簽出 Authorization Code，race condition 已經擋住了。
#
#
#
---
## 檢查 Discovery 和 JWKS 的內容，確認只允許 S256（設計文件 Step 6、Step 8）

這一步在做什麼：OpenIddict 會自動發布兩份公開文件，給 MCP Client 和 MCP Resource Server 讀：
- Discovery（/.well-known/openid-configuration）：告訴 client 這台 Server 的各個端點在哪裡、支援哪些 grant、哪些 PKCE 方法。MCP Client 第一次連線時會先讀這份。
- JWKS（/.well-known/jwks）：公開驗簽用的公鑰。MCP Resource Server 會抓這份來驗 JWT 的簽章。

這兩份是 OpenIddict 依照 Program.cs 的設定自動產生的，我們沒有寫任何程式。這一步要做的是確認它們對外宣告的內容符合設計文件，特別是 PKCE。設計文件規定只允許 S256，而 Program.cs 目前只呼叫了 RequireProofKeyForCodeExchange()，意思是「一定要帶 PKCE」，但沒有限制方法。如果 plain 也被允許，client 可以直接把 code_verifier 原文當成 code_challenge 送出，PKCE 的保護就沒有意義了。

這一步先不改程式，看完結果再決定。

1. **讀 Discovery**
GET https://localhost:7280/.well-known/openid-configuration

逐項確認回傳的 JSON：
{% asset_image openid-configuration.jpg openid-configuration%}

2. **讀 JWKS**
GET https://localhost:7280/.well-known/jwks

確認 keys 裡的那把公鑰：
- kty 是 RSA，alg 是 RS256。
- kid 和你之前在 jwt.io 解開 access_token 時 header 裡的 kid 一樣。Resource Server 就是用 kid 找出該用哪一把公鑰驗簽的。
{% asset_image kid_check.jpg kid_check%}

3. 測 plain 會不會被接受
用 GET 送出下面的網址，Follow redirects 關掉：

https://localhost:7280/connect/authorize?client_id=pnp-mcp-client&redirect_uri=https%3A%2F%2Flocalhost%3A5055%2Fcallback&response_type=code&scope=ucs.sn-info.read&code_challenge=abcdefghijklmnopqrstuvwxyzabcdefghijklmnopq&code_challenge_method=plain

- 如果被拒絕：預期是 400，或是導回 redirect_uri 並帶上 error=invalid_request。代表 OpenIddict 本來就只接受 S256，不需要改程式。
- 如果回 302 導向 localhost:16888/sso/bridge：代表 plain 被接受了，下一步要在 Program.cs 把它關掉。


**這個測試要回答的問題是：MCP Client 能不能用比較弱的 plain 方法做 PKCE？ 設計文件規定只能用 S256。**

先複習 PKCE 在做什麼

<span style="color: red;">PKCE 是為了防止「Authorization Code 被偷走後，被別人拿去換 token」：</span>

1. Client 先產生一個只有自己知道的隨機字串，叫 <span style="color: red;">code_verifier</span>。
2. 呼叫 /connect/authorize 時，送出由它算出來的 <span style="color: red;">code_challenge</span>。
3. 拿到 code 之後，呼叫 /connect/token 時要附上原本的 code_verifier。Server 驗證它和當初的 code_challenge 對得上，才發 token。

就算有人在半路偷到 code，手上沒有 code_verifier 也換不到 token。

S256 和 plain 的差別
{% asset_image diff_of_s256_plain.jpg diff_of_s256_plain %}

code_challenge 會出現在 /connect/authorize 的網址上，可能留在瀏覽器歷史紀錄、proxy log、Referer 裡。用 plain 的話，等於把通關密碼寫在網址上，PKCE 就失去保護作用了。所以 OAuth 2.1 和設計文件都規定只能用 S256。

這個測試在做什麼

Program.cs 目前只寫了 RequireProofKeyForCodeExchange()，意思是「一定要帶 PKCE」，但沒有規定只能用哪一種方法。我不確定 OpenIddict 預設允不允許 plain，所以用實際請求來確認：故意送一個 code_challenge_method=plain 的請求，看 Server 的反應。

結果 A：被拒絕（400，或是導回 redirect_uri 並帶上 error=invalid_request）
- OpenIddict 在請求進到我們的程式之前，就判定 plain 不支援。
- 代表預設就只接受 S256，符合設計文件，不用改程式。

結果 B：回 302，導向 localhost:16888/sso/bridge
- 代表 OpenIddict 接受了 plain，請求一路進到我們的 Authorize()，開始走 Bridge 流程。導向 Bridge 是「第一次經過」的正常行為，你之前也看過這個網址。
- 這就違反設計文件了，要在 Program.cs 加設定把 plain 關掉。

我們的測試結果回應的是[302]。
{% asset_image S256_TEST.jpg S256_TEST %}

<span style="color: red;">因為OAuth強制使用S256，所以我們要把plain移除：</span>
Program.cs 的 .AddServer(...)：在 RequireProofKeyForCodeExchange() 下面加上
``` csharp
// 強制所有 client 都要用 PKCE（不只 public client）
options.RequireProofKeyForCodeExchange();
// AllowAuthorizationCodeFlow() 預設會同時開啟 plain 與 S256 兩種 PKCE 方法；
// plain 的 code_challenge 就是 code_verifier 原文，等於沒有保護，依 OAuth2.1 與設計文件只保留 S256
options.Configure(o => o.CodeChallengeMethods.Remove(CodeChallengeMethods.Plain));
```

移除後，code_challenge_methods_supported 只剩 S256，對外宣告會正確。重新測試api後(code_challenge_method = plain)，也會返回[400]
{% asset_image remove_plain.jpg remove_plain %}
{% asset_image remove_plain2.jpg remove_plain2 %}


**PKCE 現在只接受 S256，Discovery 的宣告和實際行為也一致了，設計文件 Step 6、8 的這部分確認完成。**
#
#
#
---
## 加固 bridge-callback，讓它和 /connect/authorize 第二次經過一樣嚴謹

這一步在做什麼：前兩步加固的是「第二次經過 /connect/authorize」，但前一站 BridgeCallback() 還有兩個同類型的漏洞：

1. 身分資料可以被覆蓋：目前只檢查 pending「沒用過、沒過期」，沒有檢查「是不是已經回填過身分」。同一個 nonce 收到第二次 bridge-callback 時，會用新的 Redeem 結果蓋掉前一次的 UserName。如果攻擊者拿到受害者的 nonce，就可以用自己的 bridgeCode 把受害者那筆 pending 換成攻擊者的身分，受害者最後拿到的會是攻擊者帳號的 token（這叫 login CSRF）。而且這裡用的是「讀 → 改 → 存」的寫法，跟上一步一樣有 race condition。
2. 沒有輸入驗證：bridgeCode、nonce 沒帶、太長、或含 null byte（%00），都會直接往下走。SECURITY.md 要求所有新端點都要做 null byte 防護。

解法和上一步相同：先驗證輸入，再用一條 UPDATE ... WHERE "UserName" IS NULL 原子性地回填，保證每個 nonce 只能回填一次。
BridgeCallback()：整個方法換成下面這樣
``` csharp
[HttpGet("~/connect/authorize/bridge-callback")]
public async Task<IActionResult> BridgeCallback([FromQuery] string? bridgeCode, [FromQuery] string? nonce)
{
    // 輸入驗證（比照SECURITY.md）：必填、長度上限、null byte防護
    if (!IsValidCallbackParameter(bridgeCode) || !IsValidCallbackParameter(nonce))
    {
        _logger.LogWarning("BridgeCallback rejected. Reason=InvalidParameter.");
        return BadRequest("缺少或格式錯誤的參數。");
    }

    var pending = await _db.PendingAuthorizeRequests.AsNoTracking().FirstOrDefaultAsync(p => p.Nonce == nonce);

    // 先確認nonce有效、而且還沒回填過身分，才去呼叫Redeem，避免浪費PNP的一次性code
    if (pending is null || pending.Used || pending.ExpiresAt < DateTimeOffset.UtcNow || pending.UserName is not null)
    {
        _logger.LogWarning("BridgeCallback rejected. Reason=InvalidOrExpiredNonce.");
        return BadRequest("無效或已過期的 nonce。");
    }

    //用bridgeCode換回PNP的身分資料{userName, siteCode, tokenI
    var (userName, siteCode, tokenId, ticketExpiration) = await RedeemAsync(bridgeCode!);

    // 用單一條UPDATE原子性地「確認尚未回填 + 寫入身分資料」：
    // 同一個nonce收到多次bridge-callback時，只有第一次能寫入
    // 注意：這裡先不標記 Used，等真正完成SignIn時才標記，避免Redeem成功但後面失敗時整筆被浪費
    var now = DateTimeOffset.UtcNow;
    var filled = await _db.PendingAuthorizeRequests
        .Where(p => p.Nonce == nonce && !p.Used && p.ExpiresAt > now && p.UserName == null)
        .ExecuteUpdateAsync(s => s
            .SetProperty(p => p.UserName, userName)
            .SetProperty(p => p.SiteCode, siteCode)
            .SetProperty(p => p.TokenId, (int?)tokenId)
            .SetProperty(p => p.TicketExpiration, (DateTimeOffset?)ticketExpiration));

    if (filled == 0)
    {
        _logger.LogWarning("BridgeCallback rejected. Reason=AlreadyFilledOrExpired.");
        return BadRequest("無效或已過期的 nonce。");
    }

    _logger.LogInformation(
        "BridgeCallback succeeded. UserName={UserName}, SiteCode={SiteCode}, TokenId={TokenId}.",
        userName, siteCode, tokenId);

    var original = JsonSerializer.Deserialize<PendingAuthorizeData>(pending.OriginalRequestJson)!;

    // ...（下面組 queryParams、Redirect 的部分維持不變）
}
```

新增一個 private 方法
``` csharp
/// <summary>
/// bridge-callback參數的基本格式檢查：必填、長度上限、不可含null byte
/// </summary>
private static bool IsValidCallbackParameter(string? value)
{
    return !string.IsNullOrEmpty(value)
        && value.Length <= 256
        && !value.Contains('\0');
}
```

幾個重點說明：

- 參數改成 string?：這個 Controller 沒有掛 [ApiController]，所以參數沒帶時 ASP.NET Core 不會自動回 400，而是傳 null 進來。標成 string? 才符合實際情況，再交給 IsValidCallbackParameter 統一檢查。
- 為什麼長度上限是 256：我們的 nonce 是 43 個字元，PNP 的 bridgeCode 也會是類似長度的隨機字串，256 已經很寬鬆。設上限是為了避免有人送超長字串進 log 或 DB 查詢。
- 讀取時加了 pending.UserName is not null 的檢查：已經回填過的 pending 會在呼叫 Redeem 之前就被擋下。等到 PNP 真的實作好，bridgeCode 是一次性的，被擋下的請求就不會白白用掉一個 code。
- 讀完之後還要用 ExecuteUpdateAsync 再判斷一次：讀取的檢查只能擋住「依序」重送的請求。兩個請求同時進來時，可能都讀到 UserName == null，一起通過檢查。最後真正把關的是 UPDATE 的 WHERE "UserName" IS NULL，只有一個請求能讓影響筆數是 1。
- (int?)tokenId、(DateTimeOffset?)ticketExpiration 的轉型：entity 裡這兩個欄位是 nullable，SetProperty 的兩個參數型別要一致，明確轉型可以避免編譯器推斷不出型別。
- 原本的 pending.UserName = ... 和 SaveChangesAsync() 都拿掉了：改由 ExecuteUpdateAsync 直接寫 DB。pending 加了 AsNoTracking()，因為現在只用來讀 OriginalRequestJson。

測試時要注意：每次測試都要先打一次 /connect/authorize 拿到新的 nonce，從 302 的 Location（導向 16888 那個網址）抄下來。

1. 輸入驗證：用同一個新 nonce 分別送出下面兩個請求，這兩個都不會回填身分，所以同一個 nonce 還可以拿來做第 2 項：
   - 沒帶 bridgeCode：https://localhost:7280/connect/authorize/bridge-callback?nonce=<你的nonce>
   {% asset_image no_bridge_code.jpg no_bridge_code %}
   - 含 null byte：https://localhost:7280/connect/authorize/bridge-callback?bridgeCode=abc%00def&nonce=<你的nonce>   
   {% asset_image null_byte.jpg null_byte %}

2. 不能覆蓋：用同一個 nonce 依序送出兩次正常請求，Follow redirects 關掉：
   - ...bridge-callback?bridgeCode=first&nonce=<你的nonce>
   - ...bridge-callback?bridgeCode=second&nonce=<你的nonce>   
   <span style="color: red;">資料表PendingAuthorizeRequests.UserName已經於第一次請求填入test\mockuser。因無法覆蓋，回應 [400]無效或已過期的 nonce。</span>
   {% asset_image invalid_nonce.jpg invalid_nonce %}

   <span style="color: red;">從頭執行一次用bridge_code換token的流程，確認正常流程沒被改壞。</span>
   {% asset_image test_success.jpg test_success %}

3. 平行請求：重新打 /connect/authorize 拿一個新的 nonce，用 curl 同時送 5 次，確認沒有<span style="color: red;">race condition</span>：
``` bash
$url = 'https://localhost:7280/connect/authorize/bridge-callback?bridgeCode=Test&nonce=PbWgH5asH5Fd-3N8pinht3EAcuHBKzHmeesQ4_-pogA'
curl.exe -k -s -Z --parallel-immediate -w "%{http_code}\n" -o NUL $url -o NUL $url -o NUL $url -o NUL $url -o NUL $url
```
{% asset_image race_condition.jpg race_condition %}

請執行後確認：
1. dotnet build 正常編譯。
2. 第 1 項兩個請求都回 400「缺少或格式錯誤的參數」，Console 有 Reason=InvalidParameter。
3. 第 2 項第一次回 302，第二次回 400，Console 有 Reason=InvalierName 是 test\mockuser，沒有被蓋掉。第一次的 Location可以走完整個流程換到 token。
4. 第 3 項剛好一個 302、四個 400。四個 400 的 log 會混著出現 InvalidOrExpiredNonce 和 AlreadyFilledOrExpired 兩種：
   - InvalidOrExpiredNonce：讀取時就被擋下。
   - AlreadyFilledOrExpired：讀取時通過，最後在 UPDATE 被擋下。

   如果你看到後面這種，就是 race condition 真的發生了，而且有被擋住。