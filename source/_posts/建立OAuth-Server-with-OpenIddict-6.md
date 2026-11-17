---
title: 建立OAuth Server with OpenIddict(6)
date: 2026-11-16 16:37:39
categories:
  - 後端開發
tags:
  - OAuth
  - OpenIddict
---
##### 接續上一篇...
##### 此篇主要再把Client註冊的部分，抽出交由OAuth.Registration處理。
正式環境不跑 seed(DEV環境的Client測試資料，在seed產生並存入DB)，所以需要一個工具，讓管理員依照「新增 client 申請流程」（設計文件第 9.2 節）核准之後，把資料寫進正式 DB：
<!-- more -->

我們的作法是：用一個 JSON 檔描述要註冊的內容，由工具讀檔後寫入 DB。
- 每個環境一份 JSON，可以放進版本控制，申請、核准都有紀錄。
- 重複執行結果相同：已存在就更新成檔案的內容，不存在就新增。
- 先檢查所有內容（例如 client 綁定的身分來源存不存在、redirect_uri 是不是 HTTPS），全部通過才寫入，而且包在同一個交易裡，不會只寫一半。
- 開發環境的 seed 之後也可以改成讀一份 registration.Development.json，把寫死在程式碼裡的 localhost 網址移出來。
#
#
#
---
先來看看為什麼建議現階段用 JSON，等到真的有需要時，再在同一套邏輯上加網頁。
{% asset_image client_manage.jpg client_manage %}

為什麼現在適合 JSON

1. 符合設計文件的方向：第 9.2 節定的是「Client 註冊採手動管理，建立申請流程文件；不實作 Dynamic Client Registration」。JSON 加 git 就是這個流程最直接的實作。
2. 規模小：目前只有 1 個身分來源、3 個 scope、1 個 client。短期內就算增加，也是個位數，異動頻率很低。
3. 不增加攻擊面：Authorization Server 是整個系統安全的核心。管理網頁一旦有漏洞，攻擊者就能新增自己的 client，或修改 redirect_uri，衝擊比一般網站大得多。

什麼時候該考慮網頁

設計文件第 1 節的限制條件 4 提到，將來可能擴充成「共用身分驗證服務」，讓其他部門也接進來。如果真的走到這一步，例如：
- 十幾個以上的 client，分屬不同部門。
- 各部門希望自己管理 client，不想每次都等管理員改檔案。

這時就值得做管理網頁。
#
#
#
---
## 定義註冊檔格式與驗證規則，建立 OAuth.Registration 專案（只驗證，不寫 DB）
這一步在做什麼：註冊檔由人編寫，最容易出錯的是打錯欄位名稱、網址寫成 HTTP、client 引用了不存在的 scope。這些錯誤都要在寫入 DB 之前全部找出來、一次列出，讓管理員修正後再執行。

註冊檔的設計原則

- 一個環境一份，例如 registration.Production.json，描述該環境完整的身分來源、scope、client，放進版本控制。
- 檔案要自成一體：client 引用的身分來源和 scope，必須定義在同一份檔案裡（offline_access 例外，它是 OAuth 標準 scope）。這樣驗證時不需要連 DB，也能確保檔案完整描述了這個環境。
- 只新增、更新，不刪除：檔案裡沒有的項目不會被刪掉。刪除的影響太大，要另外手動處理。

1. Authorization Server：新增 Registration/RegistrationManifest.cs
``` csharp
using System.Text.Json;
using System.Text.Json.Serialization;
using OAuth.AuthorizationServer.Data;

namespace OAuth.AuthorizationServer.Registration;

/// <summary>
/// 註冊檔（registration.*.json）的內容：某個環境要註冊的身分來源、scope、client（設計文件第9.2節）
/// </summary>
public class RegistrationManifest
{
    public List<IdentityProviderRegistration> IdentityProviders { get; set; } = [];
    public List<ScopeRegistration> Scopes { get; set; } = [];
    public List<ClientRegistration> Clients { get; set; } = [];

    private static readonly JsonSerializerOptions SerializerOptions = new(JsonSerializerDefaults.Web)
    {
        Converters = { new JsonStringEnumConverter() },   // "mode": "Bridge" 以字串表示
        ReadCommentHandling = JsonCommentHandling.Skip,     // 允許在註冊檔裡寫註解
        AllowTrailingCommas = true,
        UnmappedMemberHandling = JsonUnmappedMemberHandling.Disallow,   // 欄位名稱打錯（如 redirectUri）直接報錯，不會被默默忽略
    };

    public static RegistrationManifest Load(string path)
    {
        using var stream = File.OpenRead(path);
        return JsonSerializer.Deserialize<RegistrationManifest>(stream, SerializerOptions)
            ?? throw new JsonException("註冊檔內容為空。");
    }
}

public class IdentityProviderRegistration
{
    public string? ProviderId { get; set; }
    public IdentityProviderMode Mode { get; set; }
    public string? BridgeUrl { get; set; }
    public string? RedeemApiUrl { get; set; }
    public string? RevocationCheckApiUrl { get; set; }
    public string? HmacSecretRef { get; set; }
}

public class ScopeRegistration
{
    public string? Name { get; set; }
    public string? DisplayName { get; set; }
    public List<string> Resources { get; set; } = [];
}

public class ClientRegistration
{
    public string? ClientId { get; set; }
    public string? DisplayName { get; set; }
    public string? IdentityProviderId { get; set; }
    public List<string> RedirectUris { get; set; } = [];
    public List<string> Scopes { get; set; } = [];
}
```

2. Authorization Server：新增 Registration/RegistrationValidator.cs
``` csharp
using OAuth.AuthorizationServer.Data;
using static OpenIddict.Abstractions.OpenIddictConstants;

namespace OAuth.AuthorizationServer.Registration;

/// <summary>
/// 寫入資料庫前檢查註冊檔：一次列出所有錯誤，全部通過才允許寫入
/// </summary>
public static class RegistrationValidator
{
    public static List<string> Validate(RegistrationManifest manifest)
    {
        var errors = new List<string>();

        // ---------- 身分來源 ----------
        var providerIds = new HashSet<string>(StringComparer.Ordinal);
        foreach (var p in manifest.IdentityProviders)
        {
            var name = $"identityProviders[{p.ProviderId}]";

            if (string.IsNullOrWhiteSpace(p.ProviderId))
            {
                errors.Add("identityProviders：有一筆缺少 providerId。");
                continue;
            }
            if (!providerIds.Add(p.ProviderId))
            {
                errors.Add($"{name}：providerId 重複。");
            }

            // 目前只實作 Bridge 模式（設計文件第6節）
            if (p.Mode != IdentityProviderMode.Bridge)
            {
                errors.Add($"{name}：目前只支援 Bridge 模式。");
                continue;
            }

            RequireHttpsUrl(errors, name, "bridgeUrl", p.BridgeUrl);
            RequireHttpsUrl(errors, name, "redeemApiUrl", p.RedeemApiUrl);
            RequireHttpsUrl(errors, name, "revocationCheckApiUrl", p.RevocationCheckApiUrl);

            if (string.IsNullOrWhiteSpace(p.HmacSecretRef))
            {
                errors.Add($"{name}：缺少 hmacSecretRef。");
            }
        }

        // ---------- scope ----------
        var scopeNames = new HashSet<string>(StringComparer.Ordinal);
        foreach (var s in manifest.Scopes)
        {
            var name = $"scopes[{s.Name}]";

            if (string.IsNullOrWhiteSpace(s.Name))
            {
                errors.Add("scopes：有一筆缺少 name。");
                continue;
            }
            if (s.Name is Scopes.OfflineAccess or Scopes.OpenId)
            {
                errors.Add($"{name}：OAuth/OIDC 標準 scope 由 OpenIddict 內建處理，不需註冊。");
                continue;
            }
            if (!scopeNames.Add(s.Name))
            {
                errors.Add($"{name}：name 重複。");
            }

            // 沒有 resource 的 scope 不會產生 aud，Resource Server 會拒收（設計文件第9.3節）
            if (s.Resources.Count == 0)
            {
                errors.Add($"{name}：至少要有一個 resources。");
            }
        }

        // ---------- client ----------
        var clientIds = new HashSet<string>(StringComparer.Ordinal);
        foreach (var c in manifest.Clients)
        {
            var name = $"clients[{c.ClientId}]";

            if (string.IsNullOrWhiteSpace(c.ClientId))
            {
                errors.Add("clients：有一筆缺少 clientId。");
                continue;
            }
            if (!clientIds.Add(c.ClientId))
            {
                errors.Add($"{name}：clientId 重複。");
            }

            // 引用的身分來源必須定義在同一份註冊檔
            if (string.IsNullOrWhiteSpace(c.IdentityProviderId) || !providerIds.Contains(c.IdentityProviderId))
            {
                errors.Add($"{name}：identityProviderId '{c.IdentityProviderId}' 未定義在 identityProviders。");
            }

            if (c.RedirectUris.Count == 0)
            {
                errors.Add($"{name}：至少要有一個 redirectUris。");
            }
            foreach (var uri in c.RedirectUris)
            {
                if (!IsAllowedRedirectUri(uri))
                {
                    errors.Add($"{name}：redirectUri '{uri}' 必須是 HTTPS，或 http 的 loopback 位址（127.0.0.1、[::1]、localhost），且不可含 #fragment。");
                }
            }

            if (c.Scopes.Count == 0)
            {
                errors.Add($"{name}：至少要有一個 scopes。");
            }
            foreach (var scope in c.Scopes)
            {
                // 引用的 scope 必須定義在同一份註冊檔（offline_access 為標準 scope，例外）
                if (scope != Scopes.OfflineAccess && !scopeNames.Contains(scope))
                {
                    errors.Add($"{name}：scope '{scope}' 未定義在 scopes。");
                }
            }
        }

        return errors;
    }

    private static void RequireHttpsUrl(List<string> errors, string name, string field, string? value)
    {
        if (!Uri.TryCreate(value, UriKind.Absolute, out var uri) || uri.Scheme != Uri.UriSchemeHttps)
        {
            errors.Add($"{name}：{field} 必須是 HTTPS 網址。");
        }
    }

    // OAuth 2.1：redirect_uri 必須是 HTTPS；原生應用程式（如 MCP client）可用 http 的 loopback 位址接收回呼
    private static bool IsAllowedRedirectUri(string value)
    {
        if (!Uri.TryCreate(value, UriKind.Absolute, out var uri) || !string.IsNullOrEmpty(uri.Fragment))
        {
            return false;
        }

        return uri.Scheme == Uri.UriSchemeHttps
            || (uri.Scheme == Uri.UriSchemeHttp && uri.IsLoopback);
    }
}
```

3. 建立 console 專案
在 D:\Homer\Project\OAuth 下執行：
``` bash
dotnet new console -n OAuth.Registration -f net10.0
dotnet sln OAuth.slnx add OAuth.Registration
dotnet add OAuth.Registration reference OAuth.AuthorizationServer
```

4. OAuth.Registration/Program.cs：整個檔案換成
``` csharp
using System.Text.Json;
using OAuth.AuthorizationServer.Registration;

// 正式環境註冊工具（設計文件第9.2節）：讀取註冊檔，驗證後寫入 Authorization Server 的資料庫
// 用法：dotnet run -- <註冊檔路徑>
if (args.Length != 1)
{
    Console.Error.WriteLine("用法：OAuth.Registration <註冊檔路徑>");
    return 1;
}

RegistrationManifest manifest;
try
{
    manifest = RegistrationManifest.Load(args[0]);
}
catch (Exception ex) when (ex is IOException or JsonException)
{
    Console.Error.WriteLine($"無法讀取註冊檔：{ex.Message}");
    return 1;
}

var errors = RegistrationValidator.Validate(manifest);
if (errors.Count > 0)
{
    Console.Error.WriteLine($"註冊檔驗證失敗，共 {errors.Count} 項：");
    foreach (var error in errors)
    {
        Console.Error.WriteLine($"  - {error}");
    }
    return 1;
}

Console.WriteLine($"驗證通過：身分來源 {manifest.IdentityProviders.Count} 筆、scope {manifest.Scopes.Count} 筆、client {manifest.Clients.Count} 筆。");
return 0;
```

5. 新增 OAuth.Registration/registration.Development.json
內容和目前 seed 寫入的資料相同：
``` json
{
  // 開發環境的註冊內容（與目前 Program.cs 的 SeedDataAsync 相同）
  "identityProviders": [
    {
      "providerId": "pnp",
      "mode": "Bridge",
      "bridgeUrl": "https://localhost:16888/sso/bridge",
      "redeemApiUrl": "https://localhost:16888/api/internal/sso/redeem",
      "revocationCheckApiUrl": "https://localhost:16888/api/internal/sso/token-status",
      "hmacSecretRef": "IdentityProviders:Pnp:HmacSecret"
    }
  ],
  "scopes": [
    { "name": "ucs.sn-info.read", "displayName": "SN 資料讀取", "resources": [ "urn:pnp:ucs-mcp" ] },
    { "name": "ucs.ecn-rcn-info.read", "displayName": "UCS ECN/RCN 資料讀取", "resources": [ "urn:pnp:ucs-mcp" ] },
    { "name": "pnp.L1-L2-previous-rma-info.read", "displayName": "SN自動開啟點圖 讀取", "resources": [ "urn:pnp:ucs-mcp" ] }
  ],
  "clients": [
    {
      "clientId": "pnp-mcp-client",
      "displayName": "PNP UcsService MCP Client",
      "identityProviderId": "pnp",
      "redirectUris": [ "https://localhost:5055/callback" ],
      "scopes": [ "ucs.sn-info.read", "ucs.ecn-rcn-info.read", "pnp.L1-L2-previous-rma-info.read", "offline_access" ]
    }
  ]   
}
```

幾個重點說明：

- UnmappedMemberHandling = Disallow：這是這一步最實用的設定。預設情況下，JSON 裡出現 C# 類別沒有的欄位（例如把 redirectUris 打成 redirectUri），會被默默忽略，結果 client 就沒有 redirect_uri，而且沒有任何錯誤訊息。設成 Disallow 後，打錯欄位名稱會直接報錯。
- 為什麼驗證是「收集所有錯誤」，不是遇到第一個就停：註冊檔可能有好幾個錯誤。一次全部列出，管理員可以一次修完；遇到第一個就停，要反覆執行好幾次才能修完。
- redirect_uri 為什麼允許 http 的 loopback：MCP Client（例如 Claude Desktop）是安裝在使用者電腦上的程式，它會在本機開一個臨時的 port，例如 http://127.0.0.1:5055/callback，來接收 Authorization Code。這個請求不會離開使用者的電腦，所以 OAuth 2.1 允許這種情況用 http，其他情況一律要 HTTPS。uri.IsLoopback 會判斷主機是不是 127.0.0.1、[::1] 或 localhost。
- 不允許 #fragment：OAuth 規範禁止 redirect_uri 帶 #，因為 Authorization Code 是放在 query string 上傳回的。
- 為什麼 offline_access、openid 不能放在 scopes 區塊：它們是標準 scope，OpenIddict 本身就認得。client 的 scopes 可以直接引用 offline_access。
- 回傳碼（exit code）：成功回傳 0，失敗回傳 1。將來放進部署腳本時，腳本可以依回傳碼判斷要不要繼續。

測試時要注意：這一步不連 DB，只看驗證結果。
1. 正確的註冊檔：
``` bash
dotnet run --project OAuth.Registration -- OAuth.Registration\registration.Development.json
```
{% asset_image oauth_registration_run.jpg oauth_registration_run %}

2. 故意寫錯的註冊檔：複製一份 OAuth.Registration\registration.Bad.json，內容如下：
``` json
{
  "identityProviders": [
    { "providerId": "pnp", "mode": "Bridge", "bridgeUrl": "http://pnp/sso/bridge",
      "redeemApiUrl": "https://pnp/api/internal/sso/redeem", "revocationCheckApiUrl": "https://pnp/api/internal/sso/token-status" }
  ],
  "scopes": [
    { "name": "ucs.sn-info.read", "resources": [] }
  ],
  "clients": [
    { "clientId": "bad-client", "identityProviderId": "nope",
      "redirectUris": [ "http://evil.example/callback" ],
      "scopes": [ "ucs.unknown.read", "offline_access" ] }
  ]
}
```
執行：
``` bash
dotnet run --project OAuth.Registration -- OAuth.Registration\registration.Bad.json
```
3. 欄位名稱打錯：把 registration.Bad.json 裡的 "redirectUris" 改成 "redirectUri"，再執行一次。

請執行後確認：
1. dotnet build 正常編譯。
2. 第 1 項印出「驗證通過：身分來源 1 筆、scope 3 筆、client 1 筆。」。
3. 第 2 項一次列出 6 項錯誤：
   - bridgeUrl 不是 HTTPS
   - 缺少 hmacSecretRef
   - scope 沒有 resources
   - identityProviderId 'nope' 未定義
   - redirectUri 不是 HTTPS 也不是 loopback
   - scope 'ucs.unknown.read' 未定義
4. 第 3 項印出「無法讀取註冊檔」，訊息裡提到 redirectUri 這個無法對應的欄位。
#
#
#
---
## 建立 OAuth.Core 共用專案，把資料模型、migration 和註冊邏輯搬過去

這一步在做什麼：建立一個類別庫 OAuth.Core，把「資料結構」和「註冊規則」這些與網站無關的程式碼移進去。Authorization Server 和註冊工具都改成引用它：
{% asset_image add_oauth_core.jpg add_oauth_core %}

1. 建立 OAuth.Core 專案，加入方案並安裝套件
在 D:\Homer\Project\OAuth 下執行：
``` bash
dotnet new classlib -n OAuth.Core -f net10.0
Remove-Item OAuth.Core\Class1.cs
dotnet sln OAuth.slnx add OAuth.Core

dotnet add OAuth.Core package Npgsql.EntityFrameworkCore.PostgreSQL --version 10.0.3
dotnet add OAuth.Core package OpenIddict.EntityFrameworkCore --version 7.7.2
```

2. 調整專案引用
``` bash
# Authorization Server 改為引用 Core
dotnet add OAuth.AuthorizationServer reference OAuth.Core

# 註冊工具：拿掉對網站專案的引用，改為引用 Core
dotnet remove OAuth.Registration reference OAuth.AuthorizationServer\OAuth.AuthorizationServer.csproj
dotnet add OAuth.Registration reference OAuth.Core
```

3. 搬移檔案
Data 和 Migrations 已經在 git 裡，用 git mv 搬，git 會記錄成「移動」而不是「刪除＋新增」，保留檔案歷史：
``` bash
git mv OAuth.AuthorizationServer/Data OAuth.Core/Data
git mv OAuth.AuthorizationServer/Migrations OAuth.Core/Migrations
```

註冊的兩個檔案還沒 commit，用一般的移動。順便把 Registration.cs 改名為 RegistrationValidator.cs，讓檔名和類別名稱一致：
``` bash
New-Item -ItemType Directory OAuth.Core\Registration | Out-Null
Move-Item OAuth.AuthorizationServer\Registration.cs OAuth.Core\Registration\RegistrationValidator.cs
Move-Item OAuth.AuthorizationServer\RegistrationManifest.cs OAuth.Core\Registration\RegistrationManifest.cs
```

4. 修改 namespace
搬過去的檔案，namespace 都還是 OAuth.AuthorizationServer.Data、.Migrations、.Registration，要改成 OAuth.Core.*。總共 17 個檔案（其中 migration 就有 11 個），用下面的指令一次替換：
``` bash
$utf8 = New-Object Text.UTF8Encoding($false)

# OAuth.Core 裡的檔案：namespace 與 using 全部改成 OAuth.Core.*
Get-ChildItem OAuth.Core -Recurse -Filter *.cs | Where-Object { $_.FullName -notmatch '\\(bin|obj)\\' } | ForEach-Object {
    $text = [IO.File]::ReadAllText($_.FullName)
    $new = $text.Replace('OAuth.AuthorizationServer.', 'OAuth.Core.')
    if ($new -ne $text) { [IO.File]::WriteAllText($_.FullName, $new, $utf8); "已更新 $($_.Name)" }
}

# 網站與註冊工具：只改 using
foreach ($dir in 'OAuth.AuthorizationServer', 'OAuth.Registration') {
    Get-ChildItem $dir -Recurse -Filter *.cs | Where-Object { $_.FullName -notmatch '\\(bin|obj)\\' } | ForEach-Object {
        $text = [IO.File]::ReadAllText($_.FullName)
        $new = $text.Replace('using OAuth.AuthorizationServer.Data;', 'using OAuth.Core.Data;').Replace('using OAuth.AuthorizationServer.Registration;', 'using OAuth.Core.Registration;')
        if ($new -ne $text) { [IO.File]::WriteAllText($_.FullName, $new, $utf8); "已更新 $($_.Name)" }
    }
}
```

預期會列出：
- OAuth.Core：3 個 Data 檔、11 個 Migrations 檔、2 個 Registration 檔。
- 網站：AuthorizationController.cs、Program.cs、IdentityProviderClient.cs、PendingAuthorizeRequestCleanupService.cs。
- 註冊工具：Program.cs。

5. 之後的 dotnet ef 指令寫法
DbContext 和 migration 現在在 OAuth.Core，但連線字串、設定在 Authorization Server。所以之後所有 dotnet ef 指令都要指定兩個專案：
``` bash
dotnet ef migrations add <名稱> --project OAuth.Core --startup-project OAuth.AuthorizationServer
dotnet ef database update --project OAuth.Core --startup-project OAuth.AuthorizationServer
```
<span style="color: red;">連線相關設定在OAuth.AuthorizationServer，Migration檔案在OAuth.Core。dotnet ef 會執行 OAuth.AuthorizationServer 的 Program.cs，取得連線設定；再到 OAuth.Core 讀取 migration。兩個專案你都要指定，至於裡面的檔案，它會自己找到。!!</span>

幾個重點說明：

- 為什麼用 PowerShell 的 [IO.File]::ReadAllText / WriteAllText，不用 Get-Content / Set-Content：Windows PowerShell 5.1 的 Set-Content 預設用系統的 ANSI 編碼寫檔，會把程式碼裡的中文註解變成亂碼。[IO.File] 加上指定 UTF-8 才能安全地保留中文。
幾個重點說明：

- 為什麼用 PowerShell 的 [IO.File]::ReadAllText / WriteAllText，不用 Get-Content / Set-Content：Windows PowerShell 5.1 的 Set-Content 預設用系統的 ANSI 編碼寫檔，會把程式碼裡的中文註解變成亂碼。[IO.File] 加上指定 UTF-8 才能安全地保留中文。
- migration 的 namespace 改了，DB 會不會以為是新的 migration：不會。DB 的 __EFMigrationsHistory 表只記錄 migration 的 ID，例如 20261021030433_InitialCreate，不包含 namespace。ID 寫在每個 .Designer.cs 的 [Migration("...")] 裡，這次沒有動到。
- 為什麼不用另外設定 migration 的位置：EF Core 預設把 migration 放在**DbContext 所在的組件**。DbContext 搬到 OAuth.Core，migration 也在 OAuth.Core，兩者一致，不用額外設定。
- OAuth.Core 為什麼要裝 Npgsql：migration 裡有 PostgreSQL 專用的設定（例如 Npgsql.EntityFrameworkCore.PostgreSQL.Metadata），所以 migration 所在的專案必須引用 Npgsql。Authorization Server 原本的 Npgsql、OpenIddict.EntityFrameworkCore 套件可以保留，重複引用不會出問題。
- Microsoft.EntityFrameworkCore.Design 留在 Authorization Server：dotnet ef 需要的是 startup project 裡有這個套件，也就是 --startup-project 指定的那個專案。
- IdentityProviderClient、PendingAuthorizeRequestCleanupService 留在 Authorization Server：它們分別負責「呼叫外部 API」和「網站的背景工作」，屬於網站的執行邏輯，不是共用的資料結構。

測試時要注意：這一步不改任何功能，所以要確認「搬完之後和原本完全一樣」。

1. 編譯整個方案：
``` bash
dotnet build OAuth.slnx
```
2. 確認 EF 認得搬過去的 migration，而且 DB 已經全部套用：
``` bash
dotnet ef migrations list --project OAuth.Core --startup-project OAuth.AuthorizationServer
```
3.確認 model 和 migration 的 snapshot 一致，搬移沒有讓 EF 誤以為模型變了：
``` bash
dotnet ef migrations has-pending-model-changes --project OAuth.Core --startup-project OAuth.AuthorizationServer
```
4. 啟動 Authorization Server 和 MockPnp，跑一次完整流程。
5. 執行註冊工具的驗證：
``` bash
dotnet run --project OAuth.Registration -- OAuth.Registration\registration.Development.json
```

請執行後確認：
1. 第 1 項編譯成功，沒有錯誤。
2. 第 2 項列出 5 個 migration（InitialCreate 到 AddProviderIdToPendingAuthorizeRequest），而且都沒有標示 (Pending)。
3. 第 3 項顯示沒有待處理的 model 變更（No changes have been made to the model since the last migration.）。
4. 第 4 項完整流程正常。
5. 第 5 項印出「驗證通過」。
6. git status 裡，Data、Migrations 的檔案顯示為 renamed（R），不是一堆刪除和新增。
#
#
#
---
## 註冊工具寫入 DB，並把連線字串移出 appsettings.json
這一步在做什麼：
1. OAuth.Core 統一「怎麼連 DB」：新增 AddAuthorizationServerDb(...)，網站和註冊工具都用它註冊 DbContext 和 OpenIddict Core。
2. OAuth.Core 新增寫入邏輯 RegistrationService：依註冊檔新增或更新身分來源、scope、client，全部包在同一個交易裡。
3. 註冊工具連上 DB：驗證通過後呼叫 RegistrationService，印出每一筆是「新增」還是「更新」。
4. 連線字串改放 user-secrets：網站和註冊工具各自從自己的 user-secrets 讀取，appsettings.json 裡不再有密碼。

1. OAuth.Core：新增 ServiceCollectionExtensions.cs。<span style="color: red;">讓OAuth.AuthorizationServer/Program.cs跟OAuth.Registration/Program.cs可以共用，不用重複寫</span>
``` csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using OAuth.Core.Data;

namespace OAuth.Core;

public static class ServiceCollectionExtensions
{
    /// <summary>
    /// 註冊 Authorization Server 的資料庫與 OpenIddict Core（Application/Authorization/Scope/Token 的管理服務）。
    /// 連線字串由呼叫端（網站、註冊工具）從各自的設定讀取後傳入
    /// </summary>
    public static IServiceCollection AddAuthorizationServerDb(this IServiceCollection services, string connectionString)
    {
        services.AddDbContext<AuthorizationServerDbContext>(options => options.UseNpgsql(connectionString));

        services.AddOpenIddict()
            .AddCore(options =>
            {
                // 告訴 OpenIddict 用我們的 DbContext 存 Applications/Authorizations/Scopes/Tokens
                options.UseEntityFrameworkCore()
                       .UseDbContext<AuthorizationServerDbContext>();
            });

        return services;
    }
}
```

2. OAuth.Core：新增 Registration/RegistrationService.cs
``` csharp
using System.Text.Json;
using OAuth.Core.Data;
using OpenIddict.Abstractions;
using static OpenIddict.Abstractions.OpenIddictConstants;

namespace OAuth.Core.Registration;

/// <summary>
/// 依註冊檔新增或更新身分來源、scope、client（只新增/更新，不刪除）。
/// 全部在同一個交易內完成，任何一筆失敗就全部還原，不會只寫入一半
/// </summary>
public class RegistrationService
{
    private readonly AuthorizationServerDbContext _db;
    private readonly IOpenIddictApplicationManager _applicationManager;
    private readonly IOpenIddictScopeManager _scopeManager;

    public RegistrationService(
        AuthorizationServerDbContext db,
        IOpenIddictApplicationManager applicationManager,
        IOpenIddictScopeManager scopeManager)
    {
        _db = db;
        _applicationManager = applicationManager;
        _scopeManager = scopeManager;
    }

    /// <summary>
    /// 呼叫前必須先通過 RegistrationValidator.Validate
    /// </summary>
    /// <returns>每一筆的處理結果（新增/更新），供操作者確認與留存</returns>
    public async Task<List<string>> ApplyAsync(RegistrationManifest manifest)
    {
        var results = new List<string>();

        // OpenIddict 的 manager 與這裡使用同一個 DbContext（同一個 DI scope），所以會一起納入這個交易
        await using var transaction = await _db.Database.BeginTransactionAsync();

        // ---------- 身分來源 ----------
        foreach (var p in manifest.IdentityProviders)
        {
            var entity = await _db.IdentityProviders.FindAsync(p.ProviderId);
            if (entity is null)
            {
                entity = new IdentityProvider { ProviderId = p.ProviderId! };
                _db.IdentityProviders.Add(entity);
                results.Add($"新增 身分來源 {p.ProviderId}");
            }
            else
            {
                results.Add($"更新 身分來源 {p.ProviderId}");
            }

            entity.Mode = p.Mode;
            entity.BridgeUrl = p.BridgeUrl;
            entity.RedeemApiUrl = p.RedeemApiUrl;
            entity.RevocationCheckApiUrl = p.RevocationCheckApiUrl;
            entity.HmacSecretRef = p.HmacSecretRef;
        }
        await _db.SaveChangesAsync();

        // ---------- scope ----------
        foreach (var s in manifest.Scopes)
        {
            var descriptor = new OpenIddictScopeDescriptor
            {
                Name = s.Name,
                DisplayName = s.DisplayName,
            };
            descriptor.Resources.UnionWith(s.Resources);

            var existing = await _scopeManager.FindByNameAsync(s.Name!);
            if (existing is null)
            {
                await _scopeManager.CreateAsync(descriptor);
                results.Add($"新增 scope {s.Name}");
            }
            else
            {
                await _scopeManager.UpdateAsync(existing, descriptor);
                results.Add($"更新 scope {s.Name}");
            }
        }

        // ---------- client ----------
        foreach (var c in manifest.Clients)
        {
            // 依設計文件第9.2節：一律為 public client、自動核准（Implicit consent），只開放 authorization_code 與 refresh_token
            var descriptor = new OpenIddictApplicationDescriptor
            {
                ClientId = c.ClientId,
                ClientType = ClientTypes.Public,
                ConsentType = ConsentTypes.Implicit,
                DisplayName = c.DisplayName,
            };

            foreach (var uri in c.RedirectUris)
            {
                descriptor.RedirectUris.Add(new Uri(uri));
            }

            descriptor.Permissions.UnionWith(
            [
                Permissions.Endpoints.Authorization,
                Permissions.Endpoints.Token,
                Permissions.GrantTypes.AuthorizationCode,
                Permissions.GrantTypes.RefreshToken,
                Permissions.ResponseTypes.Code,
            ]);
            foreach (var scope in c.Scopes)
            {
                descriptor.Permissions.Add(Permissions.Prefixes.Scope + scope);
            }

            // 綁定身分來源（設計文件第6節）
            descriptor.Properties[IdentityProvider.ApplicationPropertyName] = JsonSerializer.SerializeToElement(c.IdentityProviderId);

            var existing = await _applicationManager.FindByClientIdAsync(c.ClientId!);
            if (existing is null)
            {
                await _applicationManager.CreateAsync(descriptor);
                results.Add($"新增 client {c.ClientId}");
            }
            else
            {
                await _applicationManager.UpdateAsync(existing, descriptor);
                results.Add($"更新 client {c.ClientId}");
            }
        }

        await transaction.CommitAsync();
        return results;
    }
}
```

3. OAuth.AuthorizationServer/Program.cs：改用 AddAuthorizationServerDb

最上面加上 using OAuth.Core;

把原本的：
``` csharp
builder.Services.AddDbContext<AuthorizationServerDbContext>(options =>
{
    options.UseNpgsql(connectionString);
});

builder.Services.AddOpenIddict()
    .AddCore(options =>
    {
        // 告訴 OpenIddict 用我們的 DbContext 存 Applications/Authorizations/Scopes/Tokens
        options.UseEntityFrameworkCore()
               .UseDbContext<AuthorizationServerDbContext>();
    })
    .AddServer(options =>
```
改成：
``` csharp
// 資料庫與 OpenIddict Core（與註冊工具共用同一套設定）
builder.Services.AddAuthorizationServerDb(connectionString);

builder.Services.AddOpenIddict()
    .AddServer(options =>
```
.AddServer(...) 和 .AddValidation(...) 的內容都不用改。

4. 註冊工具：安裝套件，Program.cs 加上寫入
``` bash
dotnet add OAuth.Registration package Microsoft.Extensions.Hosting
```

OAuth.Registration/Program.cs 整個檔案換成：
``` csharp
using System.Text.Json;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using OAuth.Core;
using OAuth.Core.Registration;

// 正式環境註冊工具（設計文件第9.2節）：讀取註冊檔，驗證後寫入 Authorization Server 的資料庫
// 用法：dotnet run -- <註冊檔路徑>
if (args.Length != 1)
{
    Console.Error.WriteLine("用法：OAuth.Registration <註冊檔路徑>");
    return 1;
}

RegistrationManifest manifest;
try
{
    manifest = RegistrationManifest.Load(args[0]);
}
catch (Exception ex) when (ex is IOException or JsonException)
{
    Console.Error.WriteLine($"無法讀取註冊檔：{ex.Message}");
    return 1;
}

var errors = RegistrationValidator.Validate(manifest);
if (errors.Count > 0)
{
    Console.Error.WriteLine($"註冊檔驗證失敗，共 {errors.Count} 項：");
    foreach (var error in errors)
    {
        Console.Error.WriteLine($"  - {error}");
    }
    return 1;
}

Console.WriteLine($"驗證通過：身分來源 {manifest.IdentityProviders.Count} 筆、scope {manifest.Scopes.Count} 筆、client {manifest.Clients.Count} 筆。");

var builder = Host.CreateApplicationBuilder();

// 開發環境的連線字串放在 user-secrets；正式環境由環境變數 ConnectionStrings__AuthorizationServerDb 提供
builder.Configuration.AddUserSecrets<Program>(optional: true);
builder.Logging.SetMinimumLevel(LogLevel.Warning);   // 不顯示 EF 的 SQL log，只留警告與錯誤

var connectionString = builder.Configuration.GetConnectionString("AuthorizationServerDb");
if (string.IsNullOrEmpty(connectionString))
{
    Console.Error.WriteLine("找不到連線字串 'ConnectionStrings:AuthorizationServerDb'。");
    return 1;
}

builder.Services.AddAuthorizationServerDb(connectionString);
builder.Services.AddScoped<RegistrationService>();

using var host = builder.Build();
using var scope = host.Services.CreateScope();

var results = await scope.ServiceProvider.GetRequiredService<RegistrationService>().ApplyAsync(manifest);

Console.WriteLine("寫入完成：");
foreach (var result in results)
{
    Console.WriteLine($"  - {result}");
}
Console.WriteLine("提醒：Authorization Server 會快取 client 資料，請重啟後再使用新的設定。");
return 0;
```

5. 連線字串改放 user-secrets
先從 OAuth.AuthorizationServer/appsettings.json 複製目前的連線字串，然後在 D:\Homer\Project\OAuth 下執行（字串用單引號包住，裡面的 @ 才不會被 PowerShell 解讀）：

# 網站
``` bash
dotnet user-secrets set "ConnectionStrings:AuthorizationServerDb" '<連線字串>' --project OAuth.AuthorizationServer
```

# 註冊工具
``` bash
dotnet user-secrets init --project OAuth.Registration
dotnet user-secrets set "ConnectionStrings:AuthorizationServerDb" '<連線字串>' --project OAuth.Registration
```

連線字串改好後可以確認一下是否更新成功(開頭要包含Host=)
``` bash
dotnet user-secrets list --project OAuth.AuthorizationServer
dotnet user-secrets list --project OAuth.Registration
```
最後把 OAuth.AuthorizationServer/appsettings.json 裡整個 "ConnectionStrings": { ... } 區塊刪掉。

幾個重點說明：

- 為什麼 OpenIddict Core 也放進 AddAuthorizationServerDb：註冊工具要用 IOpenIddictApplicationManager、IOpenIddictScopeManager 寫入 client 和 scope，它們屬於 OpenIddict Core。網站和註冊工具都需要，所以和 DbContext 一起放在共用的方法裡。
- 網站呼叫兩次 AddOpenIddict() 沒關係嗎：沒關係。OpenIddict 的設計允許多次呼叫，每次拿到的是同一組設定，分別加上 Core、Server、Validation 就好。
- 交易怎麼包住 OpenIddict 的寫入：RegistrationService 注入的 _db，和 OpenIddict manager 內部使用的 DbContext，在同一個 DI scope 裡是同一個實例。所以在 _db 上開的交易，OpenIddict 的 CreateAsync、UpdateAsync 也會包含在內。最後 CommitAsync() 之前有任何例外，await using 會自動還原。
- UpdateAsync(existing, descriptor) 是「以檔案為準」：它會把 DB 裡的 client 整個改成 descriptor 的內容，包括 RedirectUris、Permissions。所以從註冊檔拿掉某個 redirect_uri 後再執行一次，DB 裡的也會被拿掉。只有「整個 client 從檔案裡刪掉」時，DB 裡的那筆才不會被刪除。
- 為什麼要提醒重啟 Authorization Server：之前測試身分來源綁定時發現過，OpenIddict 會把 client 資料快取在記憶體裡。註冊工具改了 DB，執行中的網站不一定馬上看得到。
- Host.CreateApplicationBuilder() 不傳 args：args 會被當成設定來源（例如 --Key=Value）。我們的參數是註冊檔路徑，不是設定值，所以不傳進去。
- AddUserSecrets<Program>(optional: true)：一般 console 程式只有在環境設成 Development 時才會讀 user-secrets。這裡明確加上，開發時就不用另外設定環境。正式機上沒有 user-secrets 檔案，optional: true 讓它直接略過，改用環境變數。
- 舊的密碼還留在 git 歷史裡：刪掉 appsettings.json 的連線字串之後，之前的 commit 裡仍然看得到它。這是本機開發用的資料庫，風險不大，但建議之後換一組密碼。正式環境的 DB 密碼從一開始就不要出現在任何檔案裡。

測試時要注意：用一份測試用的註冊檔，確認「新增」和「更新」兩種情況。

1. 網站改從 user-secrets 讀連線字串：重啟 Authorization Server，跑一次完整流程。
2. 既有資料全部是「更新」：
dotnet run --project OAuth.Registration -- OAuth.Registration\registration.Development.json
3. 新增 client：複製一份 registration.Development.json，存成 OAuth.Registration\registration.Test.json，在 clients 陣列加上第二筆：
``` json
{
  "clientId": "test-client",
  "displayName": "註冊工具測試用",
  "identityProviderId": "pnp",
  "redirectUris": [ "http://127.0.0.1:8080/callback" ],
  "scopes": [ "ucs.sn-info.read" ]
}
```
   執行兩次：
``` bash   
dotnet run --project OAuth.Registration -- OAuth.Registration\registration.Test.json
```
4. 到 DB 確認 test-client 的內容：
``` sql
SELECT "ClientId", "ClientType", "ConsentType", "RedirectUris", "Permissions", "Properties"
FROM "OpenIddictApplications" WHERE "ClientId" = 'test-client';
```
5. 測完刪掉測試用的 client：
``` sql
DELETE FROM "OpenIddictApplications" WHERE "ClientId" = 'test-client';
```

請執行後確認：
1. dotnet build OAuth.slnx 正常編譯。
2. 第 1 項：appsettings.json 沒有連線字串，完整流程照樣能換到 token。
3. 第 2 項：5 筆全部是「更新」（身分來源 1、scope 3、client 1）。
4. 第 3 項：第一次執行，test-client 是「新增」，其他是「更新」；第二次執行全部是「更新」。
5. 第 4 項：test-client 是 public、implicit，RedirectUris 是 ["http://127.0.0.1:8080/callback"]，Permissions 包含 scp:ucs.sn-info.read，Properties 是 {"identity_provider_id":"pnp"}。