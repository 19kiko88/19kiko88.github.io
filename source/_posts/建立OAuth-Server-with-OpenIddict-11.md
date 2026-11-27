---
title: 建立OAuth Server with OpenIddict(11)
date: 2026-11-27 14:49:37
password: 7533967
categories:
  - 後端開發
tags:
  - OAuth
  - OpenIddict
---
##### 接續上一篇...
##### 此篇主要再把MockPnp的部分，實作到真正的PNP專案上
<!-- more -->


#
#
#
---
## POST /api/internal/sso/token-status
這一步在做什麼

Authorization Server 在 refresh_token 換新時，會呼叫這支 API 問「這個使用者在 PNP 登出了嗎？」。目前是由 MockPnp 回答，這一步讓真正的 PNP 來回答。

這也是第一次驗證「同一個 HMAC 元件、兩把不同密鑰」的設計：這支 API 用密鑰 B（Authorization Server 的那組），密鑰 A（MCP Server 的那組）打不進來。這一步不需要新資料表，只查詢既有的 SQL Server。

1. 取得密鑰 B
密鑰 B 要和 Authorization Server 目前使用的值相同（只會顯示在你自己的畫面上）：
``` bash
cd D:\Homer\Project\OAuth\OAuth.AuthorizationServer
dotnet user-secrets list | Select-String 'IdentityProviders:Pnp:HmacSecret'
```

2. 本機密鑰檔加上密鑰 B：PCBRepairNavi/InternalApi.secrets.config

PNP 會和兩個角色溝通，兩者都是對方呼叫 PNP 的內部 API，各用一把不同的 HMAC 密鑰：
1. MCP Server（Resource Server）：PNP 是資料來源。MCP Server <span style="color: red;">以密鑰 A 呼叫 PNP 的內部 API（如 ecn-rcn）</span>，PNP 回傳資料，由 MCP Server 包裝成 MCP Tool 的結果。
2. OAuth Server（Authorization Server）：PNP 是身分來源。使用者登入時，PNP 透過瀏覽器把一次性的 bridgeCode 交給 OAuth Server，OAuth Server 再<span style="color: red;">以密鑰 B 呼叫 PNP 的 redeem 換回使用者身分</span>；refresh 時以 token-status 確認使用者是否已在 PNP 登出。Authorization Code 是 OAuth Server 發給 MCP Client 的，不會經過 PNP。

在 PCBRepairNavi/InternalApi.secrets.config的 appSettings 裡加一行（原本的 InternalApi.McpHmacSecret 保留）：
``` xml
<?xml version="1.0" encoding="utf-8"?>
<appSettings>
  <add key="InternalApi.McpHmacSecret" value="<密鑰 A，維持原值>" />
  <add key="InternalApi.SsoHmacSecret" value="<步驟 1 查到的密鑰 B>" />
</appSettings>
```
改完一定要驗證 XML 格式（格式錯誤會讓整個網站無法啟動）：
``` bash
[xml](Get-Content 'D:\Homer\Project\PCBNavigatePortal\PCBRepairNavi\InternalApi.secrets.config' -Raw) | Out-Null; 'XML OK'
```


| 交換的東西 | 方向 | 說明 |
|---|---|---|
| `bridgeCode` | PNP → OAuth Server（透過瀏覽器轉址） | 使用者在 PNP 已登入，PNP 發一張一次性取貨單 |
| 身分資料（`userName`、`siteCode`、`tokenId`…） | PNP → OAuth Server（redeem，伺服器對伺服器，HMAC 密鑰 B） | OAuth Server 拿 `bridgeCode` 來換，PNP 回答「這是誰」 |
| 登出狀態（`active`） | PNP → OAuth Server（token-status，伺服器對伺服器，HMAC 密鑰 B） | refresh 時，OAuth Server 問「他登出了嗎」 |
| Authorization Code | OAuth Server → MCP Client（透過瀏覽器轉址到 `localhost:5056/callback`） | 和 PNP 無關 |

流程上的位置（docs/MCP-EndToEnd-Flow.md）：
⑦ 瀏覽器 → PNP /sso/bridge：PNP 確認已登入，發 bridgeCode → 302 回 OAuth Server（bridgeCode 給 OAuth Server）
⑧ OAuth Server → PNP redeem：用 bridgeCode 換身分資料（PNP → OAuth Server）
⑩ OAuth Server：簽發 Authorization Code → 302 到 Claude Code 的 callback（給 MCP Client，不是給 PNP）

3. 新增 PCBRepairNavi/Controllers/InternalSsoApiController.cs
用 VS 的「加入 → 新增項目」建立，舊式 csproj 才會加上 <Compile Include>。
``` csharp
using Newtonsoft.Json;
using PCBRepairNavi.Filters;
using PCBRepairNavi.Repository;
using PCBRepairNavi.Services;
using PCBRepairNavi.Services.Interfaces;
using System;
using System.Text;
using System.Web.Mvc;

namespace PCBRepairNavi.Controllers
{
    /// <summary>
    /// Authorization Server（OAuth repo）→ PNP 的 SSO 內部 API（OAuth repo docs/OAuth2.1-MCP-AuthServer-Design.md 第4.4節）。
    /// 刻意繼承 Controller 而非 BaseController：呼叫端是伺服器，沒有 .ASPXAUTH；存取控制由 [InternalApiHmac] 負責。
    /// 刻意不套用 CSRF 驗證（伺服器對伺服器），理由見 InternalApiHmacAttribute。
    /// 路徑必須在 /api/ 底下：Web.config 全站 &lt;deny users="?" /&gt;，只有 &lt;location path="api"&gt; 允許未登入的請求。
    /// 使用密鑰 B（InternalApi.SsoHmacSecret），與 MCP Server 的密鑰 A 分開：一組外洩只影響一條路徑。
    /// </summary>
    [RoutePrefix("api/internal/sso")]
    [InternalApiHmac("InternalApi.SsoHmacSecret")]
    public class InternalSsoApiController : Controller
    {
        private readonly ILogService _logService;

        public InternalSsoApiController(ILogService logService)
        {
            _logService = logService;
        }

        /// <summary>
        /// 撤銷查詢：refresh_token 換新時，Authorization Server 確認使用者是否已在 PNP 登出
        /// </summary>
        [HttpPost]
        [Route("token-status")]
        public ActionResult TokenStatus()
        {
            TokenStatusRequest req = null;
            try
            {
                string body = HttpContext.Items[InternalApiHmacAttribute.RawBodyItemKey] as string;
                req = JsonConvert.DeserializeObject<TokenStatusRequest>(body);
            }
            catch (Exception)
            {
                req = null;   // JSON 格式錯誤、數字溢位 → 400
            }

            if (req == null || !req.TokenId.HasValue || req.TokenId.Value <= 0)
            {
                Log("TokenId=- Result=BadRequest");
                return JsonStatus(400, new { message = "Bad Request" });
            }

            try
            {
                bool active = MSSQLDataContext.IsTokenActive(req.TokenId.Value);
                Log($"TokenId={req.TokenId.Value} Active={active}");
                return JsonStatus(200, new { active });
            }
            catch (Exception ex)
            {
                ErrorLog.LogError(ex, "[InternalSsoApiController.TokenStatus]");
                Log($"TokenId={req.TokenId.Value} Result=Error");
                // 查不到狀態時不可回 active=true；回非 200，Authorization Server 會拒絕 refresh（fail-closed）
                return JsonStatus(503, new { message = "Service Unavailable" });
            }
        }

        private ContentResult JsonStatus(int statusCode, object payload)
        {
            Response.StatusCode = statusCode;
            Response.TrySkipIisCustomErrors = true;
            return Content(JsonConvert.SerializeObject(payload), "application/json", Encoding.UTF8);
        }

        private void Log(string message)
        {
            try
            {
                _logService.Log("[InternalApi][sso.token-status] " + message);
            }
            catch
            {
                // log 失敗不可影響回應
            }
        }

        private class TokenStatusRequest
        {
            public int? TokenId { get; set; }
        }
    }
}
```

4. 建置，再用 tp-caevm-vt01(TEST) 發行到 /PNP

發行前再確認一次步驟 2 的 XML OK。

幾個重點說明

1. 同一個 filter，換一個設定名稱：[InternalApiHmac("InternalApi.SsoHmacSecret")]。第一階段寫的元件一行都不用改，這就是當初讓 filter 用建構子參數指定密鑰名稱的原因。
2. 密鑰 B 和 Authorization Server 是同一個值：Authorization Server 用它簽章，PNP 用它驗章。目前 MockPnp 也使用同一個值，所以之後從 MockPnp 切換到 PNP 時，Authorization Server 的密鑰不用改。
3. int? TokenId，而不是 int：用 nullable 才分得出「沒送 tokenId」（null）和「送了 0」。數字超過 int 範圍時，Json.NET 會拋例外，被 catch 轉成 400，不會變成 500（SECURITY.md §4 整數溢位的同樣精神）。
4. 查詢失敗回 503，不回 {"active": true}：這是 fail-closed（出錯時往安全的方向倒）。資料庫連不上時，寧可讓使用者重新登入，也不能讓已登出的人繼續換新 token。
5. 直接呼叫 MSSQLDataContext.IsTokenActive：這是 BaseController 防登出重播攻擊所用的同一個方法（SECURITY.md §7.2），判斷標準完全一致。它只是查詢，符合「SQL Server 只能查詢」的限制。
6. 路徑和 MockPnp 相同：/api/internal/sso/token-status 和 MockPnp、設計文件第 4.4 節完全一致。之後切換到正式 PNP 時，Authorization Server 不用改程式，只要改註冊檔的網址（交接文件第 8.6 節）。
7. 兩個 Controller 有重複的 JsonStatus：只有 5 行，先保留重複，等 redeem 加進來時再考慮抽成共用的基底類別。

測試步驟

沿用第一階段的 D:\Homer\Test\Invoke-PnpInternalApi.ps1。先設定兩把密鑰到環境變數（只對目前的視窗有效）：
``` bash
$env:PNP_SSO_HMAC_SECRET = '<密鑰 B>'
$env:PNP_MCP_HMAC_SECRET = '<密鑰 A>'
cd D:\Homer\Test
$base = 'https://tp-caevm-vt01.corpnet.asus/PNP/'
```
{% asset_image params_setting.jpg params_setting %}


| # | 指令 | 驗證什麼 |
|:---:|---|---|
| 1 | `.\Invoke-PnpInternalApi.ps1 -BaseUrl $base -RelativePath 'api/internal/sso/token-status' -Body '{"tokenId":1}' -Secret $env:PNP_SSO_HMAC_SECRET` | 密鑰 B 可以呼叫；能查詢 SQL Server 的 UserToken |
| 2 | 同 #1，但 `-Secret $env:PNP_MCP_HMAC_SECRET` | 密鑰 A 打不進 SSO 的 API，證明兩把密鑰確實分開 |
| 3 | 同 #1，但 `-Body '{"tokenId":"abc"}'` | 輸入驗證 |
| 4 | `.\Invoke-PnpInternalApi.ps1 -BaseUrl $base -Body '{"bomNo":"60NR0BV0-MB5010","userName":"ucs\\mcp-test"}' -Secret $env:PNP_MCP_HMAC_SECRET` | 第一階段的 ecn-rcn 沒有被影響（回歸測試） |


請執行後確認

- [ ] 發行前 XML OK
- [ ] 部署後，PNP 既有頁面可以正常開啟（網站有正常啟動）
- [ ] #1：HTTP/1.1 200，{"active":false}（tokenId 1 剛好還在登入中的話，也可能是 true）
- [ ] #2：HTTP/1.1 401，{"message":"Unauthorized"}
- [ ] #3：HTTP/1.1 400，{"message":"Bad Request"}
- [ ] #4：HTTP/1.1 200，{"items":[...]}
- [ ] 結果不如預期的話，請貼上 curl 的狀態列和 header（不需要貼 X-Signature），以及 log 裡對應的那幾行

測試結果
1. api [token-status] 測試
{% asset_image api_token-status_test.jpg api_token-status_test %}

2. Secret換成[PNP_MCP_HMAC_SECRET]，回應401。
{% asset_image api_token-status_test_2.jpg api_token-status_test_2 %}

3. 錯誤格式TokenId測試。
{% asset_image api_token-status_test_3.jpg api_token-status_test_3 %}

4. api [ecn-rcn] 回歸測試
{% asset_image api_token-status_test_4.jpg api_token-status_test_4 %}


