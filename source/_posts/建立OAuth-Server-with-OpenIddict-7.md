---
title: 建立OAuth Server with OpenIddict(7)
date: 2026-11-17 17:37:09
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
