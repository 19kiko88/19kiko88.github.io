---
title: 建立OAuth Server with OpenIddict(7)
date: 2026-11-17 17:37:09
password: 7533967
categories:
  - 後端開發
tags:
  - OAuth
  - OpenIddict
---
##### 單元測試
<!-- more -->

#
#
#
---
## 建立 OAuth.Tests 測試專案，用自動化測試跑一次完整流程
這一步在做什麼：用程式模擬你之前在 Insomnia 和 PowerShell 做的每一個動作。測試會在記憶體裡啟動整個 Authorization Server，不用開 port、不用另外跑 MockPnp。
要讓測試能自己跑起來，需要準備三樣東西：
{% asset_image test_items.jpg test_items %}

1. 建立測試專案
在 D:\Homer\Project\OAuth 下執行：
``` bash
dotnet new xunit -n OAuth.Tests -f net10.0
dotnet sln OAuth.slnx add OAuth.Tests
dotnet add OAuth.Tests reference OAuth.AuthorizationServer
dotnet add OAuth.Tests package Microsoft.AspNetCore.Mvc.Testing --version 10.0.3
Remove-Item OAuth.Tests\UnitTest1.cs
```

2. 測試資料庫的連線字串放在測試專案的 user-secrets
資料庫名稱一定要包含 test，測試程式會檢查，避免誤刪開發資料庫：
``` bash
dotnet user-secrets init --project OAuth.Tests
dotnet user-secrets set "ConnectionStrings:AuthorizationServerDbTest" 'Host=localhost;Port=5432;Database=OAuth_Test;Username=postgres;Password=<你的密碼>' --project OAuth.Tests
```
OAuth_Test 資料庫不用先建，測試會自動建立。

3. 讓測試專案看得到網站的 Program
在 OAuth.AuthorizationServer/Program.cs 的最後面加上一行：
``` csharp
// 讓測試專案（WebApplicationFactory<Program>）可以存取 Program
public partial class Program { }
```

4. 新增 OAuth.Tests/Infrastructure/TestDatabase.cs
``` csharp
using Microsoft.Extensions.Configuration;
using Npgsql;

namespace OAuth.Tests.Infrastructure;

/// <summary>
/// 讀取測試資料庫的連線字串（測試專案的 user-secrets 或環境變數）
/// </summary>
public static class TestDatabase
{
    public static string GetConnectionString()
    {
        var configuration = new ConfigurationBuilder()
            .AddUserSecrets(typeof(TestDatabase).Assembly, optional: true)
            .AddEnvironmentVariables()
            .Build();

        var connectionString = configuration.GetConnectionString("AuthorizationServerDbTest")
            ?? throw new InvalidOperationException("找不到測試資料庫連線字串 'ConnectionStrings:AuthorizationServerDbTest'。");

        // 測試會刪除並重建資料庫：資料庫名稱必須包含 test，避免誤刪開發或正式資料庫
        var database = new NpgsqlConnectionStringBuilder(connectionString).Database;
        if (database is null || !database.Contains("test", StringComparison.OrdinalIgnoreCase))
        {
            throw new InvalidOperationException($"測試資料庫名稱必須包含 'test'，目前是 '{database}'。");
        }

        return connectionString;
    }
}
```

5. 新增 OAuth.Tests/Infrastructure/TestCertificates.cs
``` csharp
using System.Security.Cryptography;
using System.Security.Cryptography.X509Certificates;

namespace OAuth.Tests.Infrastructure;

/// <summary>
/// 測試用的簽章/加密憑證：測試執行時臨時產生 .pfx，供 Authorization Server 依設定載入
/// </summary>
public static class TestCertificates
{
    public static string CreatePfx(string directory, string name, X509KeyUsageFlags usage, string password)
    {
        using var rsa = RSA.Create(2048);
        var request = new CertificateRequest($"CN={name}", rsa, HashAlgorithmName.SHA256, RSASignaturePadding.Pkcs1);
        request.CertificateExtensions.Add(new X509KeyUsageExtension(usage, critical: true));

        using var certificate = request.CreateSelfSigned(DateTimeOffset.UtcNow.AddMinutes(-5), DateTimeOffset.UtcNow.AddDays(1));

        var path = Path.Combine(directory, $"{name}.pfx");
        File.WriteAllBytes(path, certificate.Export(X509ContentType.Pfx, password));
        return path;
    }
}
```

6. 新增 OAuth.Tests/Infrastructure/FakePnpHandler.cs
``` csharp
using System.Collections.Concurrent;
using System.Net;
using System.Net.Http.Json;
using System.Security.Cryptography;
using System.Text;
using System.Text.Json;

namespace OAuth.Tests.Infrastructure;

/// <summary>
/// 取代真正的 PCBRepairNavi：攔截 Authorization Server 送出的 redeem、token-status 請求（設計文件第4.4節），
/// 驗證 HMAC 簽章後直接在記憶體回應
/// </summary>
public class FakePnpHandler : HttpMessageHandler
{
    private readonly string _hmacSecret;
    private readonly ConcurrentDictionary<string, IssuedCode> _codes = new();
    private readonly ConcurrentDictionary<int, bool> _loggedOut = new();
    private int _nextTokenId;

    public FakePnpHandler(string hmacSecret)
    {
        _hmacSecret = hmacSecret;
    }

    /// <summary>
    /// 模擬 /sso/bridge：視為使用者已登入，發一個綁定 nonce 的一次性 bridgeCode
    /// </summary>
    public string IssueBridgeCode(string nonce, TimeSpan? ticketLifetime = null)
    {
        var code = Convert.ToHexString(RandomNumberGenerator.GetBytes(32));
        var tokenId = Interlocked.Increment(ref _nextTokenId);
        _codes[code] = new IssuedCode(nonce, tokenId, DateTimeOffset.UtcNow.Add(ticketLifetime ?? TimeSpan.FromHours(12)));
        return code;
    }

    /// <summary>
    /// 模擬使用者在 PNP 登出/被踢
    /// </summary>
    public void Logout(int tokenId) => _loggedOut[tokenId] = true;

    protected override async Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
    {
        var body = request.Content is null ? "" : await request.Content.ReadAsStringAsync(cancellationToken);
        if (!VerifySignature(request, body))
        {
            return new HttpResponseMessage(HttpStatusCode.Unauthorized);
        }

        using var json = JsonDocument.Parse(body);
        switch (request.RequestUri!.AbsolutePath)
        {
            case "/api/internal/sso/redeem":
                var code = json.RootElement.GetProperty("code").GetString()!;
                var nonce = json.RootElement.GetProperty("nonce").GetString();
                // 取出即作廢，並且 code 必須是為這個 nonce 發的
                if (!_codes.TryRemove(code, out var issued) || issued.Nonce != nonce)
                {
                    return new HttpResponseMessage(HttpStatusCode.BadRequest);
                }
                return JsonResponse(new
                {
                    userName = "test\\mockuser",
                    siteCode = "AS",
                    tokenId = issued.TokenId,
                    ticketExpiration = issued.TicketExpiration.ToUnixTimeSeconds(),
                });

            case "/api/internal/sso/token-status":
                var tokenId = json.RootElement.GetProperty("tokenId").GetInt32();
                return JsonResponse(new { active = !_loggedOut.ContainsKey(tokenId) });

            default:
                return new HttpResponseMessage(HttpStatusCode.NotFound);
        }
    }

    // 與 MockPnp 相同的簽章格式：timestamp\nmethod\npath\nsha256(body)
    private bool VerifySignature(HttpRequestMessage request, string body)
    {
        if (!request.Headers.TryGetValues("X-Timestamp", out var timestamp)
            || !request.Headers.TryGetValues("X-Signature", out var signature))
        {
            return false;
        }

        var bodyHash = Convert.ToHexStringLower(SHA256.HashData(Encoding.UTF8.GetBytes(body)));
        var canonical = $"{timestamp.Single()}\n{request.Method.Method}\n{request.RequestUri!.AbsolutePath}\n{bodyHash}";
        var expected = Convert.ToBase64String(HMACSHA256.HashData(Encoding.UTF8.GetBytes(_hmacSecret), Encoding.UTF8.GetBytes(canonical)));

        return expected == signature.Single();
    }

    private static HttpResponseMessage JsonResponse(object value) => new(HttpStatusCode.OK) { Content = JsonContent.Create(value) };

    private record IssuedCode(string Nonce, int TokenId, DateTimeOffset TicketExpiration);
}
```

7. 新增 OAuth.Tests/Infrastructure/AuthServerFactory.cs
``` csharp
using System.Security.Cryptography.X509Certificates;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.AspNetCore.TestHost;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using OAuth.AuthorizationServer.Services;
using OAuth.Core.Data;
using OAuth.Core.Registration;

namespace OAuth.Tests.Infrastructure;

/// <summary>
/// 在記憶體中啟動 Authorization Server：測試資料庫、臨時憑證、FakePnpHandler 取代真正的 PNP
/// </summary>
public class AuthServerFactory : WebApplicationFactory<Program>, IAsyncLifetime
{
    private const string HmacSecret = "test-hmac-secret";
    private const string CertificatePassword = "test-password";

    private readonly string _connectionString = TestDatabase.GetConnectionString();
    private readonly string _certificateDirectory = Directory.CreateTempSubdirectory("oauth-tests-").FullName;

    public FakePnpHandler Pnp { get; } = new(HmacSecret);

    public AuthServerFactory()
    {
        // 用環境變數提供設定：Program.cs 在 builder.Build() 之前就會讀取連線字串與憑證，
        // 環境變數在 WebApplication.CreateBuilder 時就已載入，確保讀得到
        Environment.SetEnvironmentVariable("ConnectionStrings__AuthorizationServerDb", _connectionString);
        Environment.SetEnvironmentVariable("IdentityProviders__Pnp__HmacSecret", HmacSecret);
        Environment.SetEnvironmentVariable("Certificates__Signing__0__Path",
            TestCertificates.CreatePfx(_certificateDirectory, "signing", X509KeyUsageFlags.DigitalSignature, CertificatePassword));
        Environment.SetEnvironmentVariable("Certificates__Signing__0__Password", CertificatePassword);
        Environment.SetEnvironmentVariable("Certificates__Encryption__0__Path",
            TestCertificates.CreatePfx(_certificateDirectory, "encryption", X509KeyUsageFlags.KeyEncipherment, CertificatePassword));
        Environment.SetEnvironmentVariable("Certificates__Encryption__0__Password", CertificatePassword);
    }

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        // 不用 Development：避免載入開發者自己的 user-secrets（開發 DB 的連線字串），也不會自動執行 seed
        builder.UseEnvironment("Testing");

        builder.ConfigureTestServices(services =>
        {
            // 呼叫 PNP 的 HttpClient 改送到 FakePnpHandler
            services.AddHttpClient<IdentityProviderClient>()
                .ConfigurePrimaryHttpMessageHandler(() => Pnp)
                .SetHandlerLifetime(Timeout.InfiniteTimeSpan);
        });
    }

    public async Task InitializeAsync()
    {
        // 重建測試資料庫並套用所有 migration（在啟動 Server 之前）
        var options = new DbContextOptionsBuilder<AuthorizationServerDbContext>().UseNpgsql(_connectionString).Options;
        await using (var db = new AuthorizationServerDbContext(options))
        {
            await db.Database.EnsureDeletedAsync();
            await db.Database.MigrateAsync();
        }

        // 用開發環境的註冊檔建立身分來源、scope、client（與網站開發環境的 seed 相同）
        using var scope = Services.CreateScope();
        var contentRoot = scope.ServiceProvider.GetRequiredService<IWebHostEnvironment>().ContentRootPath;
        var manifest = RegistrationManifest.Load(Path.Combine(contentRoot, "registration.Development.json"));
        await scope.ServiceProvider.GetRequiredService<RegistrationService>().ApplyAsync(manifest);
    }

    async Task IAsyncLifetime.DisposeAsync()
    {
        await base.DisposeAsync();
        Directory.Delete(_certificateDirectory, recursive: true);
    }
}

/// <summary>
/// 所有測試類別共用同一個 Server 與測試資料庫，並且依序執行（避免同時重建資料庫）
/// </summary>
[CollectionDefinition(Name)]
public class AuthServerCollection : ICollectionFixture<AuthServerFactory>
{
    public const string Name = "AuthServer";
}
```

8. 新增 OAuth.Tests/Infrastructure/OAuthTestClient.cs
``` csharp
using System.Net;
using System.Net.Http.Json;
using System.Security.Cryptography;
using System.Text;
using System.Text.Json;
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.AspNetCore.WebUtilities;

namespace OAuth.Tests.Infrastructure;

/// <summary>
/// 模擬瀏覽器與 MCP client 走授權流程的各個步驟（順序與手動測試時相同）
/// </summary>
public class OAuthTestClient
{
    public const string ClientId = "pnp-mcp-client";
    public const string RedirectUri = "https://localhost:5055/callback";
    public const string Scope = "ucs.sn-info.read offline_access";

    private static readonly Uri BaseAddress = new("https://localhost");

    private readonly FakePnpHandler _pnp;

    public OAuthTestClient(AuthServerFactory factory)
    {
        // 關掉自動跳轉，才能逐一檢查每個 302 的 Location；使用 https，滿足 OpenIddict 的 HTTPS 要求
        Http = factory.CreateClient(new WebApplicationFactoryClientOptions { AllowAutoRedirect = false, BaseAddress = BaseAddress });
        _pnp = factory.Pnp;
    }

    public HttpClient Http { get; }

    public static (string Verifier, string Challenge) CreatePkce()
    {
        var verifier = Base64Url(RandomNumberGenerator.GetBytes(32));
        var challenge = Base64Url(SHA256.HashData(Encoding.ASCII.GetBytes(verifier)));
        return (verifier, challenge);
    }

    public static string AuthorizeUrl(string codeChallenge, string codeChallengeMethod = "S256")
    {
        return QueryHelpers.AddQueryString("/connect/authorize", new Dictionary<string, string?>
        {
            ["client_id"] = ClientId,
            ["redirect_uri"] = RedirectUri,
            ["response_type"] = "code",
            ["scope"] = Scope,
            ["state"] = "test-state",
            ["code_challenge"] = codeChallenge,
            ["code_challenge_method"] = codeChallengeMethod,
        });
    }

    /// <summary>
    /// 第一次經過 /connect/authorize：回傳導向 Bridge 的網址上的 nonce
    /// </summary>
    public async Task<string> StartAuthorizeAsync(string codeChallenge)
    {
        var response = await Http.GetAsync(AuthorizeUrl(codeChallenge));
        Assert.Equal(HttpStatusCode.Redirect, response.StatusCode);
        return GetQueryValue(response.Headers.Location!, "nonce");
    }

    /// <summary>
    /// 模擬 PNP 導回 bridge-callback
    /// </summary>
    public Task<HttpResponseMessage> BridgeCallbackAsync(string nonce, string bridgeCode)
    {
        return Http.GetAsync(QueryHelpers.AddQueryString("/connect/authorize/bridge-callback", new Dictionary<string, string?>
        {
            ["bridgeCode"] = bridgeCode,
            ["nonce"] = nonce,
        }));
    }

    /// <summary>
    /// 跑完 authorize → bridge-callback → 第二次 authorize，回傳 Authorization Code
    /// </summary>
    public async Task<string> GetAuthorizationCodeAsync(string codeChallenge, TimeSpan? ticketLifetime = null)
    {
        var nonce = await StartAuthorizeAsync(codeChallenge);

        var callback = await BridgeCallbackAsync(nonce, _pnp.IssueBridgeCode(nonce, ticketLifetime));
        Assert.Equal(HttpStatusCode.Redirect, callback.StatusCode);

        var resume = await Http.GetAsync(callback.Headers.Location);
        Assert.Equal(HttpStatusCode.Redirect, resume.StatusCode);

        return GetQueryValue(resume.Headers.Location!, "code");
    }

    public Task<HttpResponseMessage> ExchangeCodeAsync(string code, string codeVerifier)
    {
        return Http.PostAsync("/connect/token", new FormUrlEncodedContent(new Dictionary<string, string>
        {
            ["grant_type"] = "authorization_code",
            ["code"] = code,
            ["redirect_uri"] = RedirectUri,
            ["client_id"] = ClientId,
            ["code_verifier"] = codeVerifier,
        }));
    }

    public Task<HttpResponseMessage> RefreshAsync(string refreshToken)
    {
        return Http.PostAsync("/connect/token", new FormUrlEncodedContent(new Dictionary<string, string>
        {
            ["grant_type"] = "refresh_token",
            ["refresh_token"] = refreshToken,
            ["client_id"] = ClientId,
        }));
    }

    public static async Task<JsonElement> ReadJsonAsync(HttpResponseMessage response)
    {
        return await response.Content.ReadFromJsonAsync<JsonElement>();
    }

    /// <summary>
    /// 解開 JWT 的 Payload（第二段）
    /// </summary>
    public static JsonElement DecodeJwtPayload(string jwt)
    {
        var payload = jwt.Split('.')[1].Replace('-', '+').Replace('_', '/');
        payload = payload.PadRight(payload.Length + (4 - payload.Length % 4) % 4, '=');
        return JsonDocument.Parse(Convert.FromBase64String(payload)).RootElement;
    }

    public static string GetQueryValue(Uri uri, string name)
    {
        var absolute = uri.IsAbsoluteUri ? uri : new Uri(BaseAddress, uri);
        return QueryHelpers.ParseQuery(absolute.Query)[name].ToString();
    }

    private static string Base64Url(byte[] bytes) => Convert.ToBase64String(bytes).Replace('+', '-').Replace('/', '_').TrimEnd('=');
}
```

9. 新增第一個測試 OAuth.Tests/AuthorizationFlowTests.cs
``` csharp
using System.Net;
using OAuth.Tests.Infrastructure;

namespace OAuth.Tests;

[Collection(AuthServerCollection.Name)]
public class AuthorizationFlowTests
{
    private readonly OAuthTestClient _client;

    public AuthorizationFlowTests(AuthServerFactory factory)
    {
        _client = new OAuthTestClient(factory);
    }

    [Fact]
    public async Task 完整流程_簽發含預期claim的access_token()
    {
        var (verifier, challenge) = OAuthTestClient.CreatePkce();

        var code = await _client.GetAuthorizationCodeAsync(challenge);
        var response = await _client.ExchangeCodeAsync(code, verifier);

        Assert.Equal(HttpStatusCode.OK, response.StatusCode);
        var tokens = await OAuthTestClient.ReadJsonAsync(response);
        Assert.True(tokens.TryGetProperty("refresh_token", out _));

        var payload = OAuthTestClient.DecodeJwtPayload(tokens.GetProperty("access_token").GetString()!);
        Assert.Equal("test\\mockuser", payload.GetProperty("sub").GetString());
        Assert.Equal("pnp", payload.GetProperty("idp").GetString());
        Assert.Equal("urn:pnp:ucs-mcp", payload.GetProperty("aud").GetString());
    }

    [Fact]
    public async Task Refresh_未被踢且未過期_換到新的access_token()
    {
        var (verifier, challenge) = OAuthTestClient.CreatePkce();
        var code = await _client.GetAuthorizationCodeAsync(challenge);
        var tokens = await OAuthTestClient.ReadJsonAsync(await _client.ExchangeCodeAsync(code, verifier));

        var response = await _client.RefreshAsync(tokens.GetProperty("refresh_token").GetString()!);

        Assert.Equal(HttpStatusCode.OK, response.StatusCode);
    }
}
```

幾個重點說明：

- WebApplicationFactory<Program>：ASP.NET Core 官方的整合測試工具。它會執行網站的 Program.cs，把整個 Server 跑在記憶體裡（TestServer），CreateClient() 拿到的 HttpClient 送出的請求會直接進到 Server，不經過網路。所以 middleware、OpenIddict、Controller 全部都會真的執行。
- 為什麼環境是 Testing，不是 Development：Development 會載入你自己的 user-secrets，裡面是開發 DB 的連線字串。萬一覆蓋設定沒生效，測試就會去刪你的開發 DB。用 Testing 就不會載入 user-secrets，所有設定都由測試自己提供。
- 為什麼用環境變數提供設定：Program.cs 在 builder.Build() 之前就會讀連線字串和載入憑證。環境變數在 WebApplication.CreateBuilder() 那一刻就讀進來了，一定趕得上。
- 資料庫名稱的安全檢查：EnsureDeletedAsync() 會刪除整個資料庫，所以 TestDatabase 先確認名稱包含 test，萬一 user-secrets 設錯成 OAuth，測試會直接失敗，不會刪掉開發資料庫。
- FakePnpHandler 怎麼取代 PNP：IdentityProviderClient 是透過 HttpClient 呼叫 PNP 的。ConfigurePrimaryHttpMessageHandler 把這個 HttpClient 最底層「真正把請求送出去」的元件換成 FakePnpHandler，所以請求不會送到 localhost:16888，而是直接交給 FakePnpHandler 處理。它一樣會檢查 HMAC 簽章，所以 IdentityProviderClient 的簽章邏輯也有被測到。
- [Collection]：xUnit 預設會平行執行不同的測試類別。所有測試都用同一個測試資料庫，所以要放在同一個 collection 裡，依序執行，也共用同一個 Server，不用每個類別都重建一次 DB。
- 測試方法可以用中文命名：C# 允許，測試結果列表會直接顯示，比較好讀。
- public partial class Program { }：Program.cs 用的是 top-level statements，編譯器產生的 Program 類別預設不是 public，測試專案存取不到。加上這行讓它變成 public。

測試時要注意：測試會用到本機的 PostgreSQL，請確認服務有在執行。

1. 編譯並執行測試：
dotnet test OAuth.Tests
2. 到 DB 確認多了一個 OAuth_Test 資料庫，開發用的 OAuth 資料庫沒有被動到。
{% asset_image db_oauth_test.jpg db_oauth_test%}

請執行後確認：
1. dotnet build OAuth.slnx 正常編譯。
2. dotnet test 顯示 通過! - 失敗: 0，通過: 2（或英文的 Passed!  - Failed: 0, Passed: 2）。
3. OAuth_Test 資料庫存在，裡面有 pnp-mcp-client；開發用的 OAuth 資料庫資料不變。
#
#
#
---
## 把之前手動測過的流程情境寫成自動化測試
這一步在做什麼：之前每一步改完，你都要開 Insomnia、PowerShell 手動驗證。這次把那些情境寫成測試，之後只要執行 dotnet test，幾秒內就能全部重新驗證一次。這些測試也是搬移 DB 邏輯時的安全網。
{% asset_image unit_test_case.jpg unit_test_case %}

1. OAuthTestClient.cs：新增兩個輔助方法
在 ExchangeCodeAsync 下面加上：

``` csharp
/// <summary>
/// 跑完整流程並換到 token，回傳 token 回應的 JSON
/// </summary>
public async Task<JsonElement> GetTokensAsync(TimeSpan? ticketLifetime = null)
{
    var (verifier, challenge) = CreatePkce();
    var code = await GetAuthorizationCodeAsync(challenge, ticketLifetime);

    var response = await ExchangeCodeAsync(code, verifier);
    Assert.Equal(HttpStatusCode.OK, response.StatusCode);
    return await ReadJsonAsync(response);
}
```

在 GetQueryValue 下面加上：
``` csharp
/// <summary>
/// 把網址中某個 query 參數換成新值（用來模擬竄改）
/// </summary>
public static string ReplaceQueryValue(Uri uri, string name, string newValue)
{
    var absolute = uri.IsAbsoluteUri ? uri : new Uri(BaseAddress, uri);
    var query = QueryHelpers.ParseQuery(absolute.Query).ToDictionary(p => p.Key, p => (string?)p.Value.ToString());
    query[name] = newValue;
    return QueryHelpers.AddQueryString(absolute.AbsolutePath, query);
}
```

2. AuthServerFactory.cs：新增查詢 DB 狀態的輔助方法
檔案最上面加上：
``` csharp
using OpenIddict.EntityFrameworkCore.Models;
```

在 InitializeAsync 下面加上：
``` csharp
/// <summary>
/// 取得最近一次建立的 Authorization（測試依序執行，即剛才那個測試的登入）及其底下所有 token 的狀態
/// </summary>
public async Task<(string? Status, List<string?> TokenStatuses)> GetLatestAuthorizationAsync()
{
    using var scope = Services.CreateScope();
    var db = scope.ServiceProvider.GetRequiredService<AuthorizationServerDbContext>();

    var authorization = await db.Set<OpenIddictEntityFrameworkCoreAuthorization>()
        .OrderByDescending(a => a.CreationDate)
        .FirstAsync();

    var tokenStatuses = await db.Set<OpenIddictEntityFrameworkCoreToken>()
        .Where(t => t.Authorization!.Id == authorization.Id)
        .Select(t => t.Status)
        .ToListAsync();

    return (authorization.Status, tokenStatuses);
}
```

3. AuthorizationFlowTests.cs：新增授權流程的測試
建構子改成同時保存 factory：
``` csharp
private readonly AuthServerFactory _factory;
private readonly OAuthTestClient _client;

public AuthorizationFlowTests(AuthServerFactory factory)
{
    _factory = factory;
    _client = new OAuthTestClient(factory);
}
```

在原本兩個測試下面加上：
``` csharp
[Fact]
public async Task Resume網址參數被竄改_拒絕且原始網址也作廢()
{
    var (_, challenge) = OAuthTestClient.CreatePkce();
    var nonce = await _client.StartAuthorizeAsync(challenge);
    var callback = await _client.BridgeCallbackAsync(nonce, _factory.Pnp.IssueBridgeCode(nonce));
    var resumeUrl = callback.Headers.Location!;

    // 攻擊者把 code_challenge 換成自己的
    var (_, attackerChallenge) = OAuthTestClient.CreatePkce();
    var tampered = await _client.Http.GetAsync(OAuthTestClient.ReplaceQueryValue(resumeUrl, "code_challenge", attackerChallenge));
    Assert.Equal(HttpStatusCode.BadRequest, tampered.StatusCode);

    // 原始網址也不能再用
    var original = await _client.Http.GetAsync(resumeUrl);
    Assert.Equal(HttpStatusCode.BadRequest, original.StatusCode);
}

[Fact]
public async Task 同一個Resume網址平行送出_只有一個成功()
{
    var (_, challenge) = OAuthTestClient.CreatePkce();
    var nonce = await _client.StartAuthorizeAsync(challenge);
    var callback = await _client.BridgeCallbackAsync(nonce, _factory.Pnp.IssueBridgeCode(nonce));
    var resumeUrl = callback.Headers.Location!;

    var responses = await Task.WhenAll(Enumerable.Range(0, 5).Select(_ => _client.Http.GetAsync(resumeUrl)));

    Assert.Equal(1, responses.Count(r => r.StatusCode == HttpStatusCode.Redirect));
    Assert.Equal(4, responses.Count(r => r.StatusCode == HttpStatusCode.BadRequest));
}

[Fact]
public async Task BridgeCallback同一個nonce送兩次_第二次不能覆蓋身分()
{
    var (_, challenge) = OAuthTestClient.CreatePkce();
    var nonce = await _client.StartAuthorizeAsync(challenge);

    var first = await _client.BridgeCallbackAsync(nonce, _factory.Pnp.IssueBridgeCode(nonce));
    var second = await _client.BridgeCallbackAsync(nonce, _factory.Pnp.IssueBridgeCode(nonce));

    Assert.Equal(HttpStatusCode.Redirect, first.StatusCode);
    Assert.Equal(HttpStatusCode.BadRequest, second.StatusCode);
}

[Fact]
public async Task BridgeCode是為別的nonce發的_Redeem失敗()
{
    var (_, challenge) = OAuthTestClient.CreatePkce();
    var nonce = await _client.StartAuthorizeAsync(challenge);

    // 這個 bridgeCode 是為另一個登入流程發的
    var otherCode = _factory.Pnp.IssueBridgeCode("another-nonce");
    var response = await _client.BridgeCallbackAsync(nonce, otherCode);

    Assert.Equal(HttpStatusCode.BadRequest, response.StatusCode);
}

[Fact]
public async Task PKCE使用plain_不會導向Bridge()
{
    var response = await _client.Http.GetAsync(OAuthTestClient.AuthorizeUrl("plain-challenge-plain-challenge-plain-challenge", "plain"));

    var redirectedToBridge = response.StatusCode == HttpStatusCode.Redirect
        && response.Headers.Location!.ToString().Contains("/sso/bridge");
    Assert.False(redirectedToBridge);
}

[Fact]
public async Task Discovery_PKCE只宣告S256()
{
    var discovery = await OAuthTestClient.ReadJsonAsync(await _client.Http.GetAsync("/.well-known/openid-configuration"));

    var methods = discovery.GetProperty("code_challenge_methods_supported").EnumerateArray().Select(m => m.GetString());
    Assert.Equal(["S256"], methods);
}
```

4. 新增 OAuth.Tests/RefreshTokenTests.cs
``` csharp
using System.Net;
using OAuth.Tests.Infrastructure;
using static OpenIddict.Abstractions.OpenIddictConstants;
using OpenIddict.Server;

namespace OAuth.Tests;

[Collection(AuthServerCollection.Name)]
public class RefreshTokenTests
{
    private readonly AuthServerFactory _factory;
    private readonly OAuthTestClient _client;

    public RefreshTokenTests(AuthServerFactory factory)
    {
        _factory = factory;
        _client = new OAuthTestClient(factory);
    }

    [Fact]
    public async Task PNP已登出_拒絕並撤銷整條鏈()
    {
        var tokens = await _client.GetTokensAsync();
        var tokenId = int.Parse(OAuthTestClient.DecodeJwtPayload(tokens.GetProperty("access_token").GetString()!)
            .GetProperty("token_id").GetString()!);

        _factory.Pnp.Logout(tokenId);
        var response = await _client.RefreshAsync(tokens.GetProperty("refresh_token").GetString()!);

        Assert.Equal(HttpStatusCode.BadRequest, response.StatusCode);
        Assert.Equal(Errors.InvalidGrant, (await OAuthTestClient.ReadJsonAsync(response)).GetProperty("error").GetString());
        await AssertLatestChainRevokedAsync();
    }

    [Fact]
    public async Task TicketExpiration已過期_拒絕並撤銷整條鏈()
    {
        // PNP ticket 在一分鐘前就已到期
        var tokens = await _client.GetTokensAsync(ticketLifetime: TimeSpan.FromMinutes(-1));

        var response = await _client.RefreshAsync(tokens.GetProperty("refresh_token").GetString()!);

        Assert.Equal(HttpStatusCode.BadRequest, response.StatusCode);
        Assert.Equal(Errors.InvalidGrant, (await OAuthTestClient.ReadJsonAsync(response)).GetProperty("error").GetString());
        await AssertLatestChainRevokedAsync();
    }

    [Fact]
    public async Task 重用已兌換的RefreshToken_被拒絕()
    {
        var tokens = await _client.GetTokensAsync();
        var oldRefreshToken = tokens.GetProperty("refresh_token").GetString()!;

        var first = await _client.RefreshAsync(oldRefreshToken);
        Assert.Equal(HttpStatusCode.OK, first.StatusCode);

        // 舊的 refresh_token 已經換過一次，再用就是重用
        var reused = await _client.RefreshAsync(oldRefreshToken);
        Assert.Equal(HttpStatusCode.BadRequest, reused.StatusCode);
    }

    private async Task AssertLatestChainRevokedAsync()
    {
        var (status, tokenStatuses) = await _factory.GetLatestAuthorizationAsync();

        Assert.Equal(Statuses.Revoked, status);
        Assert.NotEmpty(tokenStatuses);
        Assert.All(tokenStatuses, s => Assert.Equal(Statuses.Revoked, s));
    }
}
```


幾個重點說明：

- 測試的結構：每個測試都分成三段，這是常見的 Arrange / Act / Assert（準備 → 執行 → 驗證）寫法：
  - 準備：跑到需要的狀態，例如拿到 token、拿到 resume 網址。
  - 執行：做這個測試要檢查的那個動作。
  - 驗證：用 Assert 檢查結果。
- Task.WhenAll 平行送出：和之前用 curl.exe -Z 一樣，5 個請求同時送出。TestServer 在記憶體裡也會平行處理它們，所以 race condition 一樣測得到。
- ticketLifetime: TimeSpan.FromMinutes(-1)：讓 FakePnpHandler 發出一個「一分鐘前就已經到期」的 ticket。之前手動測試要把 MockPnp 設成 1 分鐘、再等 1 分鐘，這裡不用等。
- 怎麼確認「整條鏈被撤銷」：直接查測試 DB，找到這次登入的 Authorization，確認它和底下所有 token 的 Status 都是 revoked，和之前手動到 DB 查的方式一樣。因為同一個 collection 的測試是依序執行的，「最新的一筆 Authorization」一定是剛才那個測試的。
- plain 那個測試為什麼不直接斷言 400：OpenIddict 拒絕時，有可能回 400，也有可能導回 redirect_uri 並帶上 error=invalid_request。我們真正要確認的是「不會進到我們的流程、導向 Bridge」，所以就斷言這件事。
- Statuses.Revoked、Errors.InvalidGrant：OpenIddict 定義的常數，值分別是 "revoked"、"invalid_grant"。用常數比寫字串不容易拼錯。

測試時要注意：

1. 執行全部測試：
dotnet test OAuth.Tests
2. 確認測試真的能抓到問題：暫時把 AuthorizationController.cs 裡 MatchesOriginalRequest(...) 那段參數比對的 if 註解掉，再執行一次 dotnet test。測完記得改回來。

請執行後確認：
1. 第 1 項：通過: 11，失敗: 0（上一個步驟的 2 個，加上這次的 9 個）。
2. 第 2 項：Resume網址參數被竄改_拒絕且原始網址也作廢 這個測試失敗，其他測試仍然通過。這證明測試真的能抓到「參數比對被拿掉」這種錯誤。
#
#
#
---
## 註冊規則的單元測試，以及資安加固的測試
這一步在做什麼：前兩步的測試都是整合測試，會啟動整個 Server、連測試 DB。這次加兩類：
{% asset_image unit_test_case_2.jpg unit_test_case_2 %}

1. 新增 OAuth.Tests/RegistrationValidatorTests.cs
``` csharp
using System.Text.Json;
using OAuth.Core.Data;
using OAuth.Core.Registration;

namespace OAuth.Tests;

/// <summary>
/// RegistrationValidator 的單元測試：不需要啟動 Server，也不需要資料庫
/// </summary>
public class RegistrationValidatorTests
{
    // 一份完全合法的註冊檔，每個測試只修改要驗證的那個欄位
    private static RegistrationManifest CreateValidManifest() => new()
    {
        IdentityProviders =
        [
            new IdentityProviderRegistration
            {
                ProviderId = "pnp",
                Mode = IdentityProviderMode.Bridge,
                BridgeUrl = "https://pnp.example/sso/bridge",
                RedeemApiUrl = "https://pnp.example/api/internal/sso/redeem",
                RevocationCheckApiUrl = "https://pnp.example/api/internal/sso/token-status",
                HmacSecretRef = "IdentityProviders:Pnp:HmacSecret",
            },
        ],
        Scopes =
        [
            new ScopeRegistration { Name = "ucs.sn-info.read", Resources = ["urn:pnp:ucs-mcp"] },
        ],
        Clients =
        [
            new ClientRegistration
            {
                ClientId = "client",
                IdentityProviderId = "pnp",
                RedirectUris = ["https://client.example/callback"],
                Scopes = ["ucs.sn-info.read", "offline_access"],
            },
        ],
    };

    [Fact]
    public void 合法的註冊檔_沒有錯誤()
    {
        Assert.Empty(RegistrationValidator.Validate(CreateValidManifest()));
    }

    [Theory]
    [InlineData("https://client.example/callback", true)]
    [InlineData("http://127.0.0.1:5055/callback", true)]
    [InlineData("http://localhost:5055/callback", true)]
    [InlineData("http://[::1]:5055/callback", true)]
    [InlineData("http://evil.example/callback", false)]
    [InlineData("https://client.example/callback#fragment", false)]
    [InlineData("not-a-url", false)]
    public void RedirectUri_只允許HTTPS或http的loopback(string redirectUri, bool allowed)
    {
        var manifest = CreateValidManifest();
        manifest.Clients[0].RedirectUris = [redirectUri];

        var errors = RegistrationValidator.Validate(manifest);

        Assert.Equal(allowed, errors.Count == 0);
    }

    [Fact]
    public void 身分來源網址不是HTTPS_報錯()
    {
        var manifest = CreateValidManifest();
        manifest.IdentityProviders[0].BridgeUrl = "http://pnp.example/sso/bridge";

        Assert.Contains(RegistrationValidator.Validate(manifest), e => e.Contains("bridgeUrl"));
    }

    [Fact]
    public void Client引用未定義的身分來源_報錯()
    {
        var manifest = CreateValidManifest();
        manifest.Clients[0].IdentityProviderId = "nope";

        Assert.Contains(RegistrationValidator.Validate(manifest), e => e.Contains("'nope'"));
    }

    [Fact]
    public void Client引用未定義的scope_報錯()
    {
        var manifest = CreateValidManifest();
        manifest.Clients[0].Scopes = ["ucs.unknown.read"];

        Assert.Contains(RegistrationValidator.Validate(manifest), e => e.Contains("'ucs.unknown.read'"));
    }

    [Fact]
    public void Scope沒有resources_報錯()
    {
        var manifest = CreateValidManifest();
        manifest.Scopes[0].Resources = [];

        Assert.Contains(RegistrationValidator.Validate(manifest), e => e.Contains("resources"));
    }

    [Fact]
    public void 註冊OAuth標準scope_報錯()
    {
        var manifest = CreateValidManifest();
        manifest.Scopes.Add(new ScopeRegistration { Name = "offline_access", Resources = ["urn:pnp:ucs-mcp"] });

        Assert.Contains(RegistrationValidator.Validate(manifest), e => e.Contains("offline_access"));
    }

    [Fact]
    public void 多個錯誤_一次全部列出()
    {
        var manifest = CreateValidManifest();
        manifest.IdentityProviders[0].BridgeUrl = "http://pnp.example/sso/bridge";
        manifest.Clients[0].IdentityProviderId = "nope";
        manifest.Clients[0].Scopes = ["ucs.unknown.read"];

        Assert.Equal(3, RegistrationValidator.Validate(manifest).Count);
    }

    [Fact]
    public void 註冊檔欄位名稱打錯_無法讀取()
    {
        var path = Path.GetTempFileName();
        try
        {
            // redirectUris 打成 redirectUri
            File.WriteAllText(path, """{ "clients": [ { "clientId": "client", "redirectUri": [ "https://client.example/callback" ] } ] }""");

            Assert.Throws<JsonException>(() => RegistrationManifest.Load(path));
        }
        finally
        {
            File.Delete(path);
        }
    }
}
```

2. 新增 OAuth.Tests/SecurityTests.cs
``` csharp
using System.Net;
using OAuth.Tests.Infrastructure;

namespace OAuth.Tests;

/// <summary>
/// 資安加固（設計文件第9.5節）的整合測試
/// </summary>
[Collection(AuthServerCollection.Name)]
public class SecurityTests
{
    private readonly OAuthTestClient _client;

    public SecurityTests(AuthServerFactory factory)
    {
        _client = new OAuthTestClient(factory);
    }

    [Fact]
    public async Task QueryString含NullByte_回400()
    {
        var response = await _client.Http.GetAsync("/connect/authorize?client_id=pnp-mcp-client%00");

        Assert.Equal(HttpStatusCode.BadRequest, response.StatusCode);
    }

    [Fact]
    public async Task Form含NullByte_回400()
    {
        var response = await _client.Http.PostAsync("/connect/token", new FormUrlEncodedContent(new Dictionary<string, string>
        {
            ["grant_type"] = "refresh_token",
            ["refresh_token"] = "abc\0def",
            ["client_id"] = OAuthTestClient.ClientId,
        }));

        Assert.Equal(HttpStatusCode.BadRequest, response.StatusCode);
    }

    [Theory]
    [InlineData("/.well-known/openid-configuration")]   // 一般回應
    [InlineData("/connect/authorize?client_id=x%00")]   // 被 null byte 擋下的 400
    public async Task 回應都帶有安全標頭(string url)
    {
        var response = await _client.Http.GetAsync(url);

        Assert.Equal("default-src 'none'; frame-ancestors 'none'; base-uri 'none'; form-action 'none'",
            response.Headers.GetValues("Content-Security-Policy").Single());
        Assert.Equal("no-referrer", response.Headers.GetValues("Referrer-Policy").Single());
        Assert.Equal("nosniff", response.Headers.GetValues("X-Content-Type-Options").Single());
    }
}
```

幾個重點說明：

- RegistrationValidatorTests 沒有 [Collection]、沒有 fixture：它不需要 Server 和 DB，xUnit 會把它和其他測試平行執行，不會互相干擾。
- CreateValidManifest() 的寫法：先建一份「完全合法」的註冊檔，每個測試只改一個欄位。測試失敗時，就能確定是被改的那個欄位造成的，不會被其他錯誤干擾。
- [Theory] + [InlineData]：同一個測試邏輯，用多組資料各跑一次。測試結果裡會顯示成 7 個獨立的測試，哪一組失敗一目了然。redirect_uri 的規則有好幾種邊界情況，用它比寫 7 個 [Fact] 簡潔。
- """..."""：C# 11 的原始字串常值，裡面的 " 不用跳脫，適合直接寫 JSON。
- FormUrlEncodedContent 裡的 "abc\0def"：\0 就是 null byte，FormUrlEncodedContent 送出時會自動編碼成 %00，和之前用 curl.exe 送的效果一樣。
- 為什麼沒有測「沒有 Server 標頭」：Server: Kestrel 是 Kestrel 加的，但 TestServer 根本不經過 Kestrel，所以測試裡永遠不會有這個標頭。就算拿掉 AddServerHeader = false，測試也一樣會通過，這種測試沒有意義。這一項要靠實際啟動 Server 來確認，之前手動測過了。

測試時要注意：

1. 執行全部測試：
dotnet test OAuth.Tests
2. 只執行單元測試，感受一下速度差異：
dotnet test OAuth.Tests --filter "FullyQualifiedName~RegistrationValidatorTests"

請執行後確認：
1. 第 1 項：通過: 30，失敗: 0。
   - 原本 11 個。
   - RegistrationValidatorTests 15 個（[Theory] 的 7 組資料各算一個）。
   - SecurityTests 4 個（[Theory] 的 2 組資料各算一個）。
2. 第 2 項：只跑 15 個，而且明顯比全部測試快很多，因為不用啟動 Server、不用重建 DB。
#
#
#
---
## 補充
{% asset_image unit_test_compare.jpg unit_test_compare%}


單元測試和整合測試的差別

- 單元測試：只測一小塊程式本身的邏輯，不碰外部資源。RegistrationValidator.Validate() 是純粹的判斷：給一個 manifest、回傳錯誤清單，跟 DB、網路都沒關係，所以適合寫單元測試。
- 整合測試：測多個元件組合在一起是否正確。例如 Store 的「原子性回填」，正確與否取決於 PostgreSQL 實際怎麼執行那條 UPDATE。如果不連真的 DB，就測不出 race condition 有沒有被擋住，所以一定要接上真的 DB。

為什麼 Store 測試不算單元測試

雖然它也是「直接呼叫一個類別的方法」，但它需要真的資料庫才能執行。只要依賴了外部資源（DB、檔案系統、網路），通常就歸類為整合測試。

端對端的那幾個，有沒有完全「端對端」

不是百分之百。有兩個地方是替身：
- PNP：用 FakePnpHandler 取代，沒有真的連到 PCBRepairNavi。
- 網路與 Kestrel：用 TestServer 在記憶體中處理請求。所以 Server 標頭那類只有 Kestrel 才會產生的行為測不到。

除此之外，middleware、OpenIddict、Controller、DB 全部都是真的。

這幾種測試的比例

常見的說法是「測試金字塔」：

        ▲   端對端整合測試：少量，最接近真實情境，但慢、失敗時較難定位
       ▲▲▲  元件整合測試：中等數量
     ▲▲▲▲▲  單元測試：大量，執行快、失敗時一看就知道是哪裡出錯

目前專案的狀況是端對端測試比較多，這對 Authorization Server 是合理的：它最重要的是「整個流程串起來是否安全、正確」，而這只有端對端測試驗證得了。之後如果有更多純邏輯的程式碼（例如參數比對、claim 組裝），可以再抽出來補單元測試。