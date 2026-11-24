---
title: 建立OAuth Server with OpenIddict(9)
date: 2026-11-20 11:14:20
password: 7533967
categories:
  - 後端開發
tags:
  - OAuth
  - OpenIddict
---
##### 供 MCP Server 呼叫的 API 的程式開發，並透過 HMAC 確認呼叫者是 MCP Server，且請求內容未被竄改。                
- 簽章的核心邏輯與單元測試
- HMAC 驗證的 MVC Filter
- 新增 POST 內部API(透過此API來呼叫真正的UCS API)
- 設定密鑰、部署到 PnpTest、用 PowerShell 模擬 MCP Server 測試
<!-- more -->
#
#
#
---
## HMAC 簽章的核心邏輯與單元測試

這一步在做什麼

把第 1 步拆成兩半：

- 這一半：寫一個純邏輯的 static class，負責計算和驗證簽章。它不碰 HttpContext，也不讀 web.config，所以可以完全用單元測試驗證。
- 下一半：再用 ActionFilter 把它接到 MVC 上。
<!-- more -->
這樣拆的原因：

- 你沒有本機 PNP，如果簽章格式和 Authorization Server／MockPnp 有一個字元不同，部署到測試機後只會看到一片 401，很難查出原因。先在本機用測試證明「算出來的簽章和 .NET 10 版完全一樣」，之後部署出問題時，就能排除格式問題。

1. 新增檔案 1：PCBRepairNavi/Helpers/InternalApiSignature.cs
``` csharp
using System;
using System.Globalization;
using System.Runtime.CompilerServices;
using System.Security.Cryptography;
using System.Text;

namespace PCBRepairNavi.Helpers
{
    /// <summary>
    /// 驗證結果。詳細原因只寫入 log，不回傳給呼叫端（SECURITY.md §2.3）
    /// </summary>
    public enum InternalApiSignatureResult
    {
        Valid,
        MissingTimestamp,
        TimestampOutOfRange,
        MissingSignature,
        InvalidSignature
    }

    /// <summary>
    /// 伺服器對伺服器內部 API 的 HMAC-SHA256 簽章（OAuth 設計文件第4.4節）。
    /// X-Timestamp = Unix 秒數；X-Signature = Base64(HMAC-SHA256(secret, "timestamp\nmethod\npath\nsha256(body)小寫16進位"))。
    /// 格式必須與 OAuth.MockPnp、Authorization Server、PCBRepairNavi.MCP 完全一致。
    /// 
    /// 簽章方：OAuth Server. MCP Server
    /// 驗章方：PNP
    /// </summary>
    public static class InternalApiSignature
    {
        public const int AllowedClockSkewSeconds = 5;

        public static string ComputeSignature(string secret, long timestamp, string method, string path, string body)
        {
            if (string.IsNullOrEmpty(secret))
            {
                throw new ArgumentException("HMAC secret is required.", nameof(secret));
            }

            string canonical = timestamp.ToString(CultureInfo.InvariantCulture) + "\n"
                             + method + "\n"
                             + path + "\n"
                             + Sha256Hex(body ?? string.Empty);

            using (var hmac = new HMACSHA256(Encoding.UTF8.GetBytes(secret)))
            {
                return Convert.ToBase64String(hmac.ComputeHash(Encoding.UTF8.GetBytes(canonical)));
            }
        }

        /// <param name="nowUnixSeconds">目前時間；由呼叫端傳入，方便單元測試固定時間</param>
        public static InternalApiSignatureResult Verify(
            string secret, string timestampHeader, string signatureHeader,
            string method, string path, string body, long nowUnixSeconds)
        {
            // NumberStyles.None：不接受正負號、空白、小數點；長度上限避免異常輸入
            if (string.IsNullOrEmpty(timestampHeader) || timestampHeader.Length > 19 ||
                !long.TryParse(timestampHeader, NumberStyles.None, CultureInfo.InvariantCulture, out long timestamp))
            {
                return InternalApiSignatureResult.MissingTimestamp;
            }

            if (Math.Abs(nowUnixSeconds - timestamp) > AllowedClockSkewSeconds)
            {
                return InternalApiSignatureResult.TimestampOutOfRange;
            }

            if (string.IsNullOrEmpty(signatureHeader))
            {
                return InternalApiSignatureResult.MissingSignature;
            }

            string expected = ComputeSignature(secret, timestamp, method, path, body);
            if (!FixedTimeEquals(Encoding.UTF8.GetBytes(expected), Encoding.UTF8.GetBytes(signatureHeader)))
            {
                return InternalApiSignatureResult.InvalidSignature;
            }

            return InternalApiSignatureResult.Valid;
        }

        private static string Sha256Hex(string value)
        {
            using (var sha = SHA256.Create())
            {
                byte[] hash = sha.ComputeHash(Encoding.UTF8.GetBytes(value));
                return BitConverter.ToString(hash).Replace("-", "").ToLowerInvariant();
            }
        }

        /// <summary>
        /// 固定時間比較（.NET Framework 沒有 CryptographicOperations.FixedTimeEquals）。
        /// 迴圈中不可提早 return；NoOptimization 避免 JIT 把迴圈最佳化成提早結束
        /// </summary>
        [MethodImpl(MethodImplOptions.NoInlining | MethodImplOptions.NoOptimization)]
        private static bool FixedTimeEquals(byte[] a, byte[] b)
        {
            if (a == null || b == null || a.Length != b.Length)
            {
                return false;
            }

            int diff = 0;
            for (int i = 0; i < a.Length; i++)
            {
                diff |= a[i] ^ b[i];
            }
            return diff == 0;
        }
    }
}
```

2. 新增檔案 2：PCBRepairNavi.Tests/InternalApiSignatureTests.cs
``` csharp
using Microsoft.VisualStudio.TestTools.UnitTesting;
using PCBRepairNavi.Helpers;

namespace PCBRepairNavi.Tests
{
    [TestClass]
    public class InternalApiSignatureTests
    {
        private const string Secret = "unit-test-secret";
        private const long Now = 1793180718;
        private const string Method = "POST";
        private const string Path = "/PNP/api/internal/ucs/ecn-rcn";
        private const string Body = "{\"bomNo\":\"60NB0000-MB0000\",\"userName\":\"mockuser\"}";

        // 以 .NET 10 的 HMACSHA256.HashData / Convert.ToHexStringLower（與 OAuth.MockPnp 相同寫法）預先算出
        private const string ExpectedSignature = "mLzSTk8tcA7AW6fLcGRz2Pt7Y0ATEbqBhNHPpiYdgbs=";

        [TestMethod]
        public void ComputeSignature_MatchesNet10Implementation()
        {
            string actual = InternalApiSignature.ComputeSignature(Secret, Now, Method, Path, Body);
            Assert.AreEqual(ExpectedSignature, actual);
        }

        [TestMethod]
        public void Verify_ValidSignature_AtSkewBoundary_ReturnsValid()
        {
            var result = InternalApiSignature.Verify(Secret, Now.ToString(), ExpectedSignature, Method, Path, Body, Now + 5);
            Assert.AreEqual(InternalApiSignatureResult.Valid, result);
        }

        [TestMethod]
        public void Verify_TimestampSixSecondsOld_ReturnsTimestampOutOfRange()
        {
            var result = InternalApiSignature.Verify(Secret, Now.ToString(), ExpectedSignature, Method, Path, Body, Now + 6);
            Assert.AreEqual(InternalApiSignatureResult.TimestampOutOfRange, result);
        }

        [TestMethod]
        public void Verify_WrongSecret_ReturnsInvalidSignature()
        {
            var result = InternalApiSignature.Verify("wrong-secret", Now.ToString(), ExpectedSignature, Method, Path, Body, Now);
            Assert.AreEqual(InternalApiSignatureResult.InvalidSignature, result);
        }

        [TestMethod]
        public void Verify_PathWithoutVirtualDirectory_ReturnsInvalidSignature()
        {
            var result = InternalApiSignature.Verify(Secret, Now.ToString(), ExpectedSignature, Method, "/api/internal/ucs/ecn-rcn", Body, Now);
            Assert.AreEqual(InternalApiSignatureResult.InvalidSignature, result);
        }

        [TestMethod]
        public void Verify_TamperedBody_ReturnsInvalidSignature()
        {
            string tampered = Body.Replace("mockuser", "admin");
            var result = InternalApiSignature.Verify(Secret, Now.ToString(), ExpectedSignature, Method, Path, tampered, Now);
            Assert.AreEqual(InternalApiSignatureResult.InvalidSignature, result);
        }
    }
}
```

幾個重點說明

1. 為什麼是 static class，不直接寫成 ActionFilter：ActionFilter 需要 HttpContext 才能跑，沒辦法在本機測試。把「怎麼算、怎麼比」抽出來成純函式，Filter 下一步只負責讀 header 和 body、讀密鑰、決定回 401。之後 SSO 端點與 Native Capability API 也都共用這一份。
2. ExpectedSignature 是跨實作的對照值：這個值是我用 .NET 10 的 HMACSHA256.HashData + Convert.ToHexStringLower 算出來的，寫法和 MockPnp 第 169～176 行相同。如果只測「自己算、自己驗」，兩邊都錯一樣的地方時測試還是會過；有這個固定值，才能證明和 Authorization Server／MCP Server 那邊的算法一致。
3. nowUnixSeconds 由外部傳入：像是讓時鐘可以「撥動」。測試可以精準放在 +5 秒（邊界，要通過）和 +6 秒（要拒絕），不會因為測試執行時剛好跨秒而時好時壞。正式呼叫時會傳 DateTimeOffset.UtcNow.ToUnixTimeSeconds()。
4. 先檢查時間戳，再算 HMAC：過期或格式錯誤的請求不用花時間算雜湊。NumberStyles.None 會拒絕 -1、 123、1e9 這類輸入，所以 timestamp 一定 ≥ 0，Math.Abs 不會溢位。
5. 固定時間比較：== 碰到第一個不同的字元就會回傳，攻擊者可以從回應時間一個字元一個字元地猜出簽章。XOR 累加不管哪裡不同，都會跑完整個迴圈。NoOptimization 是 .NET Core 內建的 FixedTimeEquals 也有用的保護。
6. Verify_PathWithoutVirtualDirectory：把第 8.3.1 節的虛擬目錄問題寫成測試，提醒之後 Filter 一定要用 Request.Url.AbsolutePath。
7. C# 7.3 相容：PNP 的 LangVersion 是 7.3，所以沒用 using var、switch expression、record 這些新語法。

測試步驟

1. 在 VS 用「加入 → 新增項目」建立這兩個檔案。PNP 和 Tests 都是舊式 csproj，直接在檔案總管新增的話，不會自動加上 <Compile Include>，檔案不會被編譯。
2. 建置方案，然後在「測試總管」執行 InternalApiSignatureTests。這一步驗證的是簽章邏輯和 .NET 10 版完全相容。
3. 刻意改壞：把 Sha256Hex 的 .ToLowerInvariant() 拿掉後重跑，再改回來。這一步驗證的是「對照值測試」真的能抓到格式不一致。
#
#
#
---
## HMAC 驗證的 MVC Filter

這一步在做什麼

把上一步的 InternalApiSignature 接到 MVC 上，做成一個 attribute。之後只要在 action 加上 [InternalApiHmac("設定名稱")]，就會先驗章，驗章失敗的請求根本進不到 action。它負責 PNP 特有的雜事：

- 讀出原始 body
- 取出含虛擬目錄的路徑
- 從設定檔讀密鑰
- 回 401 時不要被 Forms 驗證改成 302 導向登入頁
- 寫 log

1. 新增檔案：PCBRepairNavi/Filters/InternalApiHmacAttribute.cs
``` csharp
using DocumentFormat.OpenXml.Wordprocessing;
using PCBRepairNavi.Helpers;
using PCBRepairNavi.Services;
using PCBRepairNavi.Services.Interfaces;
using System;
using System.IO;
using System.Net.Http;
using System.Text;
using System.Web;
using System.Web.Configuration;
using System.Web.Mvc;

namespace PCBRepairNavi.Filters
{
    /// <summary>
    /// UCS API包裝成internal API供外部呼叫(MCP Server)
    /// 以 HTTPS + HMAC 簽章 + 時間戳 ±5 秒防護。
    /// </summary>


    /// <summary>
    /// 伺服器(MCP Server)對伺服器內部(PNP) API 的 HMAC 驗證（OAuth 設計文件第4.2、4.4節）。
    /// 注意：此類 API 刻意不套用 CSRF 驗證（SECURITY.md §3.1 的例外）——呼叫端是伺服器，
    /// 沒有、也取得不到 PNP 的 CSRF token；改以 HTTPS + HMAC 簽章 + 時間戳 ±5 秒防護。
    /// 每條呼叫路徑使用自己的密鑰，以建構子參數指定 appSettings 的 key。
    /// </summary>
    [AttributeUsage(AttributeTargets.Class | AttributeTargets.Method, AllowMultiple = false)]
    public class InternalApiHmacAttribute : FilterAttribute, IAuthorizationFilter
    {
        /// <summary>驗章通過後，原始 body 存在 HttpContext.Items 的這個 key，供 action 反序列化</summary>
        public const string RawBodyItemKey = "InternalApi.RawBody";

        private const int MaxBodyLength = 64 * 1024;

        private readonly string _secretSettingKey;

        public InternalApiHmacAttribute(string secretSettingKey)
        {
            _secretSettingKey = secretSettingKey;
        }

        public void OnAuthorization(AuthorizationContext filterContext)
        {
            HttpContextBase httpContext = filterContext.HttpContext;
            HttpRequestBase request = httpContext.Request;
            string path = request.Url.AbsolutePath;   // 含虛擬目錄(pnp|pnptest)，必須與呼叫端的 uri.AbsolutePath 一致（交接文件第8.3.1節）

            try
            {
                //不同伺服器呼叫端會各有一組Secret，透過Secret讓PNP來區分呼叫端
                //OAuth Server=> PNP, Secret: AAAAA
                //MCP Server => PNP, Secret: BBBBB
                string secret = WebConfigurationManager.AppSettings[_secretSettingKey];
                if (string.IsNullOrEmpty(secret))
                {
                    Log($"[InternalApi] Rejected. Reason=SecretNotConfigured Key={_secretSettingKey} Path={path}");
                    Reject(filterContext);
                    return;
                }

                if (request.ContentLength > MaxBodyLength)
                {
                    Log($"[InternalApi] Rejected. Reason=BodyTooLarge Path={path}");
                    Reject(filterContext);
                    return;
                }

                string body = ReadRawBody(request);
                if (body.Length > MaxBodyLength)
                {
                    Log($"[InternalApi] Rejected. Reason=BodyTooLarge Path={path}");
                    Reject(filterContext);
                    return;
                }

                long now = DateTimeOffset.UtcNow.ToUnixTimeSeconds();
                InternalApiSignatureResult result = InternalApiSignature.Verify(
                    secret,
                    request.Headers["X-Timestamp"],
                    request.Headers["X-Signature"],
                    request.HttpMethod,
                    path,
                    body,
                    now);

                if (result != InternalApiSignatureResult.Valid)
                {
                    // 只有 TimestampOutOfRange 時 X-Timestamp 避免 log injection）
                    string clientTs = result == InternalApiSignatureResult.TimestampOutOfRange
                        ? request.Headers["X-Timestamp"]
                        : "-";
                    Log($"[InternalApi] Rejected. Reason={result} Path={path} ServerTime={now} ClientTime={clientTs}");
                    Reject(filterContext);
                    return;
                }

                httpContext.Items[RawBodyItemKey] = body;
            }
            catch (Exception ex)
            {
                ErrorLog.LogError(ex, "[InternalApiHmacAttribute]");
                Reject(filterContext);
            }
        }

        private static string ReadRawBody(HttpRequestBase request)
        {
            Stream input = request.InputStream;
            input.Position = 0;
            // leaveOpen: true —— 不可關閉 InputStream，否則之後無法再讀
            using (var reader = new StreamReader(input, Encoding.UTF8, false, 1024, leaveOpen: true))
            {
                string body = reader.ReadToEnd();
                input.Position = 0;
                return body;
            }
        }

        private static void Reject(AuthorizationContext filterContext)
        {
            HttpResponseBase response = filterContext.HttpContext.Response;
            response.StatusCode = 401;
            response.TrySkipIisCustomErrors = true;               // 不讓 IIS 換成自訂錯誤頁（SECURITY.md §1.1）
            response.SuppressFormsAuthenticationRedirect = true;  // 不讓 Forms 驗證把 401 改成 302 → /Account/Login
            filterContext.Result = new ContentResult
            {
                Content = "{\"message\":\"Unauthorized\"}",       // 通用訊息，詳細原因只在 log（SECURITY.md §2.3）
                ContentType = "application/json",
                ContentEncoding = Encoding.UTF8
            };
        }

        private static void Log(string message)
        {
            try
            {
                DependencyResolver.Current.GetService<ILogService>()?.Log(message);
            }
            catch
            {
                // log 失敗不可影響回應
            }
        }
    }
}
```

幾個重點說明

1. 為什麼用 IAuthorizationFilter，不用 ActionFilterAttribute：MVC 的執行順序是「授權 filter → model binding → action filter → action」。用授權 filter，簽章沒通過就不會做 model binding，符合第 8.3 節「先驗簽、通過才反序列化」的要求。PNP 的全域 RejectNullByteRequestFilter、AntiForgeryTokenHeaderFilter 都是 action filter，會排在它後面。
2. SuppressFormsAuthenticationRedirect = true（最容易漏掉）：PNP 用 Forms 驗證（Web.config:199）。ASP.NET 的 FormsAuthenticationModule 看到 401，會偷偷改成 302 導向 /Account/Login。少了這一行，MCP Server 收到的會是登入頁的 HTML，不是 401。
3. 密鑰設定名稱由建構子傳入：這就是「格式共用、密鑰分開」的落實方式。之後 MCP 的 API 會寫 [InternalApiHmac("McpHmacSecret")]，SSO 的 API 會寫 [InternalApiHmac("SsoHmacSecret")]，同一個 filter 依設定名稱讀不同的密鑰。
4. 讀 body 後要把 Position 歸零，而且 leaveOpen: true：就像看完錄影帶要倒帶。之後 action 從 HttpContext.Items[RawBodyItemKey] 拿已經讀好的字串，不用再讀一次 stream。
5. body 大小上限 64 KB：內部 API 的 body 只有 bomNo、userName 這種小欄位。先擋掉大 body，避免有人用大量資料讓伺服器白算 SHA256。這裡檢查兩次：ContentLength 是 header 宣告的值（不可信），讀出來後再檢查實際長度。
6. log 只寫可信的值：
   - Path 是 Url.AbsolutePath，已經過編碼，不會有換行。
   - ClientTime 只在已確認是純數字時才寫入。這樣出現 TimestampOutOfRange 時，一眼就能看出兩台主機差了幾秒（問題 C）。
   - 密鑰和 X-Signature 絕對不寫。
7. 沒有在 filter 裡檢查 HTTPS：Global.asax.cs:105 已經會把非本機的 HTTP 請求導向 HTTPS。如果在 filter 裡再擋一次，萬一前面有負責 TLS 的反向代理，IsSecureConnection 會是 false，反而會擋掉合法請求。HTTPS 的強制改由呼叫端負責：之後 MCP Server 遇到非 https:// 的設定，直接拋出設定錯誤（和 Authorization Server 的做法相同）。
8. 例外一律回 401：任何意外都不能讓請求「通過」，也不能讓全域 HandleErrorAttribute 回出 HTML 錯誤頁。

測試步驟

這個 filter 的行為（401 不被改成 302、虛擬目錄路徑、讀 body）只有在真正的 IIS 上才看得出來，在本機用假物件測也測不到這些問題。所以這一步只做到編譯，實際行為等第 2 步有 API 之後，部署到測試機一起驗證，這樣只需要部署一次。

1. 在 VS 用「加入 → 新增項目」建立這個檔案（會加上 <Compile Include>），然後建置方案。這一步驗證的是 C# 7.3 相容、namespace 與參考都正確。
2. 重跑 InternalApiSignatureTests。這一步驗證的是沒有影響到上一步。

請執行後確認

- [ ] 建置成功，沒有錯誤
- [ ] PCBRepairNavi.csproj 有 <Compile Include="Filters\InternalApiHmacAttribute.cs" />
- [ ] 6 個測試仍然全部通過
#
#
#
---
## 新增 POST 內部API
``` csharp
using Newtonsoft.Json;
using PCBRepairNavi.Areas.Viewer.Dtos.API.UCS.Res;
using PCBRepairNavi.Areas.Viewer.Services.Interfaces;
using PCBRepairNavi.Filters;
using PCBRepairNavi.Services;
using PCBRepairNavi.Services.Interfaces;
using System;
using System.Collections.Generic;
using System.Text;
using System.Text.RegularExpressions;
using System.Web.Mvc;

namespace PCBRepairNavi.Controllers
{
    /// <summary>
    /// MCP Server（PCBRepairNavi.MCP）→ PNP 的內部 API。
    /// 刻意繼承 Controller 而非 BaseController：BaseController.OnAuthorization 會把沒有 .ASPXAUTH 的
    /// 伺服器呼叫導向登入頁；存取控制改由 [InternalApiHmac] 負責。
    /// 刻意不套用 CSRF 驗證（伺服器對伺服器），理由見 InternalApiHmacAttribute。
    /// </summary>
    [RoutePrefix("api/internal/ucs")]
    [InternalApiHmac(McpHmacSecretKey)]          // ← 從 action 移到 class：之後新增的 action 自動受保護
    public class InternalUcsApiController : Controller
    {
        private const string McpHmacSecretKey = "InternalApi.McpHmacSecret";

        // PNP 的 userName 格式為「網域\帳號」，例如 ucs\xxx
        private static readonly Regex UserNameRegex =
            new Regex(@"^[A-Za-z0-9_\-]{1,30}\\[A-Za-z0-9._@\-]{1,64}$", RegexOptions.Compiled);

        private readonly IUcsService _ucsService;
        private readonly ILogService _logService;

        public InternalUcsApiController(IUcsService ucsService, ILogService logService)
        {
            _ucsService = ucsService;
            _logService = logService;
        }

        [HttpPost]
        [Route("ecn-rcn")]                       // ← 完整路徑 = api/internal/ucs/ecn-rcn
        public ActionResult EcnRcn()
        {
            EcnRcnRequest req = null;
            try
            {
                string body = HttpContext.Items[InternalApiHmacAttribute.RawBodyItemKey] as string;
                req = JsonConvert.DeserializeObject<EcnRcnRequest>(body);
            }
            catch (Exception)
            {
                req = null;   // JSON 格式錯誤 → 400
            }

            if (req == null ||
                !SecurityUtils.IsSafeParam(req.BomNo, 50, allowEmpty: false) ||
                !SecurityUtils.IsStrictSafe(req.BomNo) ||
                !SecurityUtils.IsSafeParam(req.UserName, 100, allowEmpty: false) ||
                !UserNameRegex.IsMatch(req.UserName))
            {
                Audit("-", "-", "BadRequest");   // 未通過驗證的值不寫入 log
                return JsonStatus(400, new { message = "Bad Request" });
            }

            try
            {
                var apiRes = _ucsService.GetEcnAndRcnInfo(req.BomNo);
                if (apiRes == null || !apiRes.Success)
                {
                    Audit(req.UserName, req.BomNo, "UpstreamFailed " + OneLine(apiRes?.Message));
                    return JsonStatus(502, new { message = "Upstream service unavailable" });
                }

                var items = apiRes.Content ?? new List<ResUcsEcnRcnInfoDto>();
                Audit(req.UserName, req.BomNo, "OK Count=" + items.Count);
                return JsonStatus(200, new { items });
            }
            catch (Exception ex)
            {
                ErrorLog.LogError(ex, "[InternalUcsApiController.EcnRcn]");
                Audit(req.UserName, req.BomNo, "Error");
                return JsonStatus(502, new { message = "Upstream service unavailable" });
            }
        }

        private ContentResult JsonStatus(int statusCode, object payload)
        {
            Response.StatusCode = statusCode;
            Response.TrySkipIisCustomErrors = true;
            return Content(JsonConvert.SerializeObject(payload), "application/json", Encoding.UTF8);
        }

        private void Audit(string userName, string bomNo, string result)
        {
            try
            {
                _logService.Log($"[InternalApi][ucs.ecn-rcn] User={userName} BomNo={bomNo} Result={result}");
            }
            catch
            {
                // 稽核 log 失敗不可影響回應
            }
        }

        private static string OneLine(string value)
        {
            if (string.IsNullOrEmpty(value)) return "-";
            string s = value.Replace("\r", " ").Replace("\n", " ");
            return s.Length > 200 ? s.Substring(0, 200) : s;
        }

        private class EcnRcnRequest
        {
            public string BomNo { get; set; }
            public string UserName { get; set; }
        }
    }
}
```

2. RouteConfig.cs：，在 Default 之前，加上下面這一行
``` csharp
    // 啟用屬性路由：內部 API 的路徑直接寫在 Controller 上（[RoutePrefix] / [Route]）
    routes.MapMvcAttributeRoutes();
```

之後新增 API 時，只要在同一個 Controller 加一個 action：
``` csharp
    [HttpPost]
    [Route("sn-info")]                       // → api/internal/ucs/sn-info，不用改 RouteConfig
    public ActionResult SnInfo() { ... }
```        

第三階段的 SSO 則開另一個 Controller，掛不同的前綴和密鑰：
``` csharp
    [RoutePrefix("api/internal/sso")]
    [InternalApiHmac("InternalApi.SsoHmacSecret")]   // 密鑰 B
    public class InternalSsoApiController : Controller { ... }
```

幾個重點說明

1. 繼承 Controller，不繼承 BaseController（交接文件第 6 節的陷阱）：BaseController 會檢查有沒有登入，MCP Server 的請求不帶 .ASPXAUTH，會直接被 302 導到登入頁。改成由 [InternalApiHmac] 把關。
2. 不加 [SafeValidateAntiForgeryTokenHeader]：這和 SECURITY.md §3.1 衝突，是刻意的例外。原因寫在 class 註解裡，以免資安掃描或 code review 時被質疑（交接文件第 8.4 節）。
3. 路由要放在 Default 之前：MVC 由上往下比對路由，取第一個符合的。api/internal/ucs/ecn-rcn 是固定字串，不會誤吃到其他網址。
4. body 從 HttpContext.Items 拿，不用 model binding：這是 filter 驗章時讀出來的同一份原始字串。順序上保證「先驗章、後反序列化」。Json.NET 反序列化時不分大小寫，bomNo 會對應到 BomNo。
5. 輸入驗證（SECURITY.md §1.3、§12.1）：
   - bomNo：IsSafeParam（null byte、長度 50）加上 IsStrictSafe（白名單：中文、英數、-、_）。SECURITY.md §12.1 就是把 BomNumber 列為 IsStrictSafe 的適用欄位。
   - userName：格式是 網域\帳號（AccvountController.cs:884 的 "ucs\\" + userName）。
   - 雖然有 HMAC，表示請求來自 MCP Server，仍然逐欄驗證：萬一 MCP Server 本身有 bug 或被攻破，PNP 還有最後一道防線。
6. 狀態碼：
   - 400：輸入錯誤
   - 401：簽章錯誤（由 filter 回）
   - 502：UCS 失敗
   - 回應內容一律是通用訊息，不回傳 apiRes.Message 和 ex.Message（SECURITY.md §2.3）。UCS 的錯誤訊息只寫進 log，而且先去掉換行、截到 200 字，避免 log injection。
7. 查無資料也回 200：回 {"items":[]}。「這個 bomNo 沒有 ECN/RCN」是正常結果，不是錯誤。
8. 稽核 log：記錄 userName、bomNo、結果，寫進既有的 TxtLogService。只有驗證過的值才會寫入。
9. [InternalApiHmac] 放在 class 層級：這樣不會有人新增 action 時忘了加，讓 API 沒有保護就上線。一個 Controller 對應一組密鑰，「格式共用、密鑰分開」從 Controller 的分法就看得出來。
10. 路徑和程式放在一起：看 Controller 就知道網址，不用去 RouteConfig.cs 對照。
11. MapMvcAttributeRoutes() 要放在 Default 之前：否則 Default 的 {controller}/{action}/{id} 會先比對到 api/internal/ucs，結果找不到 controller，回 404。
#
#
#
---
## 設定密鑰、部署到 PnpTest、用 PowerShell 模擬 MCP Server 測試

這一步在做什麼

1. 讓 PNP 讀得到密鑰 A：Web.config 的 appSettings 加上 file="InternalApi.secrets.config"，把外部密鑰檔合併進來。filter 照原本的寫法，用 WebConfigurationManager.AppSettings 讀取，filter 程式不用改。
2. 密鑰檔不進 git，只在發行到測試機時一起帶上。
3. 部署到 PnpTest，用 PowerShell 腳本扮演 MCP Server 簽章呼叫，驗證第 1、2 步的所有程式。

以下 A、B 兩部分在本機完成，C 部署、D 測試。

---
**A. 準備密鑰（本機）**

- A-1. 修改根目錄 .gitignore（加在第 403 行 /PCBRepairNavi/App_Data/Elmah.Errors 附近）

/PCBRepairNavi/InternalApi.secrets.config

- A-2. 產生密鑰 A（PowerShell 5.1）
``` bash
    $b = New-Object byte[] 32
    [Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($b)
    [Convert]::ToBase64String($b)
```
把輸出的字串存到你的密碼管理工具，第二階段 MCP Server 還要用。不要貼給我，也不要放進任何進 git 的檔案。

- A-3. 建立密鑰檔 PCBRepairNavi/InternalApi.secrets.config
``` xml
    $secret = 'A-2. 產生的密鑰'
    $path   = 'D:\Homer\Project\PCBNavigatePortal\PCBRepairNavi\InternalApi.secrets.config'
    $xml    = '<?xml version="1.0" encoding="utf-8"?>' + "`r`n" + '<appSettings>' + "`r`n" + '  <add key="InternalApi.McpHmacSecret" value="' + $secret + '" />' + "`r`n" + '</appSettings>' + "`r`n"
    [IO.File]::WriteAllText($path, $xml, (New-Object System.Text.UTF8Encoding($false)))
    Get-Item $path | Select-Object FullName, Length
```
▎ 不要用 VS 的「加入 → 新增項目」建立這個檔案：VS 會自動在 csproj 加上一行沒有條件的 <Content>，和 B-3 衝突。

- A-4. 驗證 XML 格式（每次修改密鑰檔後都要做）
``` bash
    [xml](Get-Content $path -Raw) | Out-Null; 'XML OK'
```
[xml](Get-Content $path -Raw) | Out-Null; 'XML OK'

印出 XML OK 才可以發行。
{% asset_image xml_ok.jpg xml_ok%}
#
#
#
---
**B. 修改程式（本機）**

- 修改 PCBRepairNavi/Web.config（第 19 行）
``` xml
    <appSettings file="InternalApi.secrets.config">
```
{% asset_image appsettings.jpg appsettings %}

- B-2. 修改 PCBRepairNavi/Controllers/InternalUcsApiController.cs 的 class 註解
``` csharp
    /// <summary>
    /// MCP Server（PCBRepairNavi.MCP）→ PNP 的內部 API。
    /// 刻意繼承 Controller 而非 BaseController：BaseController.OnAuthorization 會把沒有 .ASPXAUTH 的
    /// 伺服器呼叫導向登入頁；存取控制改由 [InternalApiHmac] 負責。
    /// 刻意不套用 CSRF 驗證（伺服器對伺服器），理由見 InternalApiHmacAttribute。
    /// 路徑必須在 /api/ 底下：Web.config 全站 <deny users="?" >，只有 <location path="api"> 允許未登入的請求。
    /// 密鑰 InternalApi.McpHmacSecret 來自 Web.config <appSettings file="InternalApi.secrets.config">（不進 git）。
    /// </summary>
```

- B-3. 修改 PCBRepairNavi/PCBRepairNavi.csproj
在方案總管對 PCBRepairNavi 按右鍵 →「卸載專案」→「編輯 PCBRepairNavi.csproj」，找到任一個含有 <Content Include="..." /> 的 <ItemGroup>，在裡面加入下面這行，存檔後「重新載入專案」：
``` xml
    <!-- MCP → PNP 的 HMAC 密鑰（測試機用）。不進 git；只有 TP-CAEVM-VT01 組態且檔案存在時才發行 -->
    <Content Include="InternalApi.secrets.config" Condition="Exists('InternalApi.secrets.config') And '$(Configuration)' == 'TP-CAEVM-VT01'" />
```
#
#
#
---
**C. 部署**
先做 A-4 確認 XML OK，再用 tp-caevm-vt01(TEST)Test 發行。
#
#
#
---
**D. 測試腳本：D:\Homer\Test\Invoke-PnpInternalApi.ps1**
這個檔案放在 repo 外面，不含密鑰，密鑰從環境變數讀取。
``` ps1
    param(
        [string]$BaseUrl = 'https://tp-caevm-vt01.corpnet.asus/Pnp/',
        [string]$RelativePath = 'api/internal/ucs/ecn-rcn',
        [string]$Body = '{"bomNo":"","userName":"ucs\\mcp-test"}',
        [string]$Secret = $env:PNP_MCP_HMAC_SECRET,
        [int]$TimestampOffset = 0
    )
    $ErrorActionPreference = 'Stop'
    if ([string]::IsNullOrEmpty($Secret)) { throw 'Please set $env:PNP_MCP_HMAC_SECRET first.' }

    # 路徑取法與 MCP Server 相同：基底網址以 / 結尾、相對路徑不以 / 開頭，再取 AbsolutePath
    $uri  = New-Object System.Uri((New-Object System.Uri($BaseUrl)), $RelativePath)
    $path = $uri.AbsolutePath
    $ts   = [DateTimeOffset]::UtcNow.ToUnixTimeSeconds() + $TimestampOffset

    $utf8     = New-Object System.Text.UTF8Encoding($false)
    $sha      = [System.Security.Cryptography.SHA256]::Create()
    $bodyHash = -join ($sha.ComputeHash($utf8.GetBytes($Body)) | ForEach-Object { $_.ToString('x2') })
    $canonical = "$ts`nPOST`n$path`n$bodyHash"

    $hmac     = New-Object System.Security.Cryptography.HMACSHA256
    $hmac.Key = $utf8.GetBytes($Secret)
    $sig      = [Convert]::ToBase64String($hmac.ComputeHash($utf8.

    # body 寫成檔案再用 --data-binary 送出，確保送出的位元組與簽章時完全相同
    $tmp = [System.IO.Path]::GetTempFileName()
    [System.IO.File]::WriteAllBytes($tmp, $utf8.GetBytes($Body))

    Write-Host "URL        : $($uri.AbsoluteUri)"
    Write-Host "Path(sign) : $path"
    Write-Host "Local UTC  : $([DateTime]::UtcNow.ToString('R'))"
    Write-Host '----------------------------------------'
    try {
        curl.exe -s -i -X POST $uri.AbsoluteUri `
            -H 'Content-Type: application/json' `
            -H "X-Timestamp: $ts" `
            -H "X-Signature: $sig" `
            --data-binary "@$tmp"
    } finally {
        Remove-Item $tmp
    }
```

**為什麼選 PowerShell 腳本：**

| 理由 | 說明 |
|---|---|
| 每次自動重算 | 腳本每次執行都會取「現在」的時間、重新計算 body 雜湊和簽章 |
| 不用另外安裝工具 | PowerShell 5.1 內建 HMACSHA256、SHA256，Windows 也內建 curl.exe |
| 和 MCP Server 的邏輯相同 | 腳本的取路徑、組簽章字串的方式，就是第二階段 MCP Server 要用 C# 寫的同一套邏輯，等於先驗證了「規格」 |
| 方便測錯誤情境 | `-Secret`、`-TimestampOffset`、`-RelativePath` 這些參數，一行就能做出 #2、#3、#5 的錯誤情境 |


---

幾個重點說明

1. appSettings file=：ASP.NET 會把外部檔案的 <add> 合併進 appSettings。檔案不存在時直接忽略，所以其他人的電腦和正式環境都和現在完全一樣。
2. ⚠️ 這個做法唯一的風險：密鑰檔存在但 XML 格式錯誤時，整個網站會無法啟動。所以 A-4 的驗證是發行前的必要步驟，不是選擇性的。
3. 對既有功能的影響：
   - 密鑰的 key InternalApi.McpHmacSecret 是全新的名稱，不會覆蓋任何既有設定。
   - Web.Release.config、Web.TP-CAEVM-VT01.config 的 transform 只修改個別的 <add>，不會動到 file 屬性。
4. 更換密鑰要重新啟動網站：修改外部檔案不會像修改 Web.config 那樣自動重新啟動，要重新發行（或重新啟動應用程式集區）才會生效。
5. 密鑰檔怎麼送上測試機：
   - 放在專案根目錄、和 Web.config 同一層，不放 App_Data，因為 publish profile 設了 ExcludeApp_Data=true。
   - 發行時由 B-3 的 <Content> 帶上，而且只限 TP-CAEVM-VT01 組態，用 Release 發行到正式環境時不會帶上。
   - SkipExtraFilesOnServer=true：其他人發行時，即使他們的電腦沒有這個檔案，測試機上的密鑰檔也會保留。
6. IIS 預設禁止下載 .config：Web.config 也是靠這個保護。測試 #6 會實際驗證。
7. 測試腳本：
   - 用 --data-binary @檔案 送出，避免 PowerShell 5.1 把 JSON 的引號吃掉、導致 body 和簽章不一致。
   - 路徑用 Uri 合併後取 AbsolutePath，和第二階段 MCP Server 的寫法相同。
   - curl -i 會顯示回應的 Date:（測試機時間），用來代替 w32tm 比對時鐘。
   - -s 不跟隨轉址，收到 302 會原樣顯示。

測試步驟

本機（發行前）：

1. 建置方案，在測試總管執行 InternalApiSignatureTests。這一步確認前面的步驟沒有受影響。
2. 執行 git status。這一步確認 Web.config、.gitignore、csproj、Controller 有變更，而且看不到 InternalApi.secrets.config。

部署後，先設定密鑰（只對目前的 PowerShell 視窗有效）：
``` bash
$env:PNP_MCP_HMAC_SECRET = '<A-2 產生的字串>'
cd D:\Homer\Test
```

| # | 指令 | 驗證什麼 |
|:---:|---|---|
| 1 | `.\Invoke-PnpInternalApi.ps1 -Body '{"bomNo":"<有資料的bomNo>","userName":"ucs\\mcp-test"}'` | 整條路徑都通（虛擬目錄、/api 放行、屬性路由、讀密鑰、呼叫 UCS），並比對時鐘 |
| 2 | 同 #1，再加上 `-Secret 'wrong-secret'` | 簽章錯誤會回 401 JSON，不會變成 302 導向登入頁 |
| 3 | 同 #1，再加上 `-TimestampOffset -10` | 時間戳過期會回 401 |
| 4 | `.\Invoke-PnpInternalApi.ps1`（bomNo 為空） | 輸入驗證會回 400 |
| 5 | 同 #1，再加上 `-RelativePath 'InternalUcsApi/EcnRcn'` | 傳統路由打不到屬性路由的 action |
| 6 | 用瀏覽器開啟：<br>`https://tp-caevm-vt01.corpnet.asus/PnpTest/InternalApi.secrets.config`<br>`https://tp-caevm-vt01.corpnet.asus/PnpTest/Areas/Viewer/Content/test_log.txt` | 密鑰檔不能被下載；稽核 log 是否會被外部下載（既有問題） |


請執行後確認

- [ ] A-3 最後一行有顯示 FullName 和 Length；A-4 印出 XML OK
- [ ] 建置成功，6 個測試全部通過
- [ ] git status：看不到 InternalApi.secrets.config
- [ ] 部署後，PNP 既有頁面能正常開啟（確認網站有正常啟動）
- [ ] #1：HTTP/1.1 200，{"items":[...]}，Date 和 Local UTC 相差在 5 秒內
- [ ] #2、#3：HTTP/1.1 401，{"message":"Unauthorized"}，不是 302
- [ ] #4：HTTP/1.1 400，{"message":"Bad Request"}
- [ ] #5：HTTP/1.1 404
- [ ] #6：密鑰檔無法下載；test_log.txt 能不能下載（能的話，裡面應該有 [InternalApi][ucs.ecn-rcn] User=ucs\mcp-test ...，以及 #3 那一筆的 Reason=TimestampOutOfRange ServerTime=... ClientTime=...）
- [ ] 結果不如預期的話，請貼上 curl 的狀態列和 header（不需要貼 X-Signature），以及 log 裡對應的那幾行   
#
#
#
---
程式裡沒有任何地方直接呼叫 InternalUcsApiController。它是由 MVC 依照收到的 HTTP 請求自動建立和呼叫的，和 PNP 既有的 Controller（例如 PCBNewController）一樣：既有的 Controller 是被瀏覽器或 Angular 呼叫，<span style="color: red;">這一支則是被 D 的測試腳本呼叫（現在由腳本扮演 MCP Server，第二階段換成真正的 MCP Server）。</span>

比喻

Controller 像是公司裡的某個分機。沒有員工會「呼叫分機的程式碼」，而是外面有人撥了號碼，總機（MVC 路由）查表後把電話轉過去。測試腳本就是那個撥號的人。

請求怎麼一路走到這個 Controller

D 的測試腳本（curl.exe）
  │  POST https://tp-caevm-vt01.corpnet.asus/PnpTest/api/internal/ucs/ecn-rcn
  ▼
IIS → PnpTest 網站
  ▼
Web.config <location path="api"> 放行未登入的請求          ← 少了這個會 302 到登入頁
  ▼
MVC 路由：routes.MapMvcAttributeRoutes()
  比對 [RoutePrefix("api/internal/ucs")] + [Route("ecn-rcn")]
  → 找到 InternalUcsApiController.EcnRcn                      ← 「呼叫」就發生在這裡
  ▼
Unity 建立 InternalUcsApiController（建構子注入 IUcsService、ILogService）
  ▼
[InternalApiHmac] 驗章（OnAuthorization）                   ← 失敗 → 401，不會進 action
  ▼
EcnRcn() 執行 → _ucsService.GetEcnAndRcnInfo(bomNo) → 回傳 JSON
#
#
#
---
**做完最後的測試。表示HMAC的簽章機制在PNP這端（驗章端）是可以成功運作了!**

| 證明的事 | 由哪一項驗證 |
|---|:---:|
| 正確的簽章會放行 | #1 |
| 錯誤的密鑰會擋下 | #2 |
| 過期的時間戳會擋下 | #3 |
| 擋下時回 401 JSON，不會被改成 302 導向登入頁 | #2、#3 |
| 驗章通過後，輸入驗證仍然有效 | #4 |
| 只能從指定的路徑呼叫 | #5 |
| 密鑰檔不會外洩 | #6 |
