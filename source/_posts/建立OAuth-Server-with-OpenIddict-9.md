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