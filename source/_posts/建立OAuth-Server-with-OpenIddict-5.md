---
title: 建立OAuth Server with OpenIddict(5)
date: 2026-11-09 14:17:50
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
## 把開發用憑證換成固定的簽章和加密憑證（設計文件第 9.1、10 節）

這一步在做什麼：Program.cs 目前用的是：
options.AddDevelopmentEncryptionCertificate()
       .AddDevelopmentSigningCertificate();

OpenIddict 會在你這台電腦的 Windows 憑證存放區自動產生這兩張憑證。在你的電腦上它們是固定的，所以之前 kid 都沒變，但它有三個問題：
1. 換一台主機就不一樣：部署到正式機或多台主機時，每台各自產生一張，A 主機簽的 JWT，B 主機發布的 JWKS 裡就找不到公鑰。
2. 無法備份和輪替：設計文件第 9.1 節要求私鑰要有備援、版本控管，輪替時採「重疊式」，但開發憑證都辦不到。
3. OpenIddict 官方說明它只能用在開發環境。

這次改成：自己產生兩張憑證檔（.pfx），放在 repo 以外的地方，由設定指定路徑，密碼放在 user-secrets。將來正式環境只要把路徑和密碼改成指向金鑰管理機制提供的憑證，程式碼不用改。

兩張憑證的用途：
{% asset_image 憑證類別.jpg 憑證類別%}

- **簽章憑證**：私鑰簽章，公鑰驗證。<span style="color: red;">證明資料來源(防止資料被竄改)，主要確認身分來源跟資料的完整性。</span>
    應用場景(RS256)如下：以JWT傳輸為例
    1. OAuth Server： 使用自己的私鑰對 JWT 產生數位簽章。
    2. MCP Server： 從 OAuth Server 公開的 JWKS Endpoint（例如 /.well-known/jwks.json）取得公鑰。
    3. 驗證 JWT： MCP Server 使用公鑰驗證 JWT 的數位簽章，確認簽章有效且內容未遭竄改。

    <span style="color: red;">簽章憑證的重點不在資料加密，而在於驗證資料的完整性與簽發來源。</span>


- **加密憑證**：公鑰加密，私鑰解密。<span style="color: red;">保護資料不被偷看(防止資料被偷看)，主要確認資料的機密性。</span>
    應用場景(RSA)如下：假設 MCP Server 要傳送員工資料給 HR Server。
    1. HR Server 建立一組 RSA 金鑰。
    2. HR Server 將公鑰憑證提供給 MCP Server。
    3. MCP Server 使用 HR Server 的公鑰加密薪資資料。
    4. MCP Server 將密文傳送給 HR Server。
    5. HR Server 使用自己的私鑰解密，取得原始薪資資料。    

    <span style="color: red;">加密憑證（Encryption Certificate）的重點不在驗證資料來源，而在於保護資料的機密性，避免未經授權的人讀取資料內容。</span>

看完簽章憑證跟加密憑證的差異後，開始進行把開發用憑證換成固定的簽章和加密憑證的流程


1. 產生兩張憑證檔

先建立一個 repo 以外的資料夾，例如 D:\Homer\Secrets，然後在 PowerShell 執行。密碼請換成你自己的，兩張要記住各自的密碼：
``` bash
$dir = 'D:\Homer\Secrets'

# 簽章憑證（KeyUsage = DigitalSignature）
$cert = New-SelfSignedCertificate -Subject 'CN=OAuth AuthorizationServer Signing' -KeyAlgorithm RSA -KeyLength 2048 -KeyUsage DigitalSignature -KeyExportPolicy Exportable -CertStoreLocation 'Cert:\CurrentUser\My' -NotAfter (Get-Date).AddYears(2)
Export-PfxCertificate -Cert $cert -FilePath "$dir\oauth-signing.pfx" -Password (ConvertTo-SecureString '換成簽章憑證密碼' -AsPlainText -Force)
Remove-Item "Cert:\CurrentUser\My\$($cert.Thumbprint)"

# 加密憑證（KeyUsage = KeyEncipherment）
$cert = New-SelfSignedCertificate -Subject 'CN=OAuth AuthorizationServer Encryption' -KeyAlgorithm RSA -KeyLength 2048 -KeyUsage KeyEncipherment -KeyExportPolicy Exportable -CertStoreLocation 'Cert:\CurrentUser\My' -NotAfter (Get-Date).AddYears(2)
Export-PfxCertificate -Cert $cert -FilePath "$dir\oauth-encryption.pfx" -Password (ConvertTo-SecureString '換成加密憑證密碼' -AsPlainText -Force)
Remove-Item "Cert:\CurrentUser\My\$($cert.Thumbprint)"
```
{% asset_image pfx_created.jpg pfx_created%}

2. 設定路徑和密碼
OAuth.AuthorizationServer/appsettings.Development.json 加上路徑（路徑不是機密，可以放設定檔）：
``` json
{
  "Certificates": {
    "Signing": { "Path": "D:\\Homer\\Secrets\\oauth-signing.pfx" },
    "Encryption": { "Path": "D:\\Homer\\Secrets\\oauth-encryption.pfx" }
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Trace"
    }
  }
}
```

密碼放 user-secrets（在 D:\Homer\Project\OAuth 下執行）：
``` bash
dotnet user-secrets set "Certificates:Signing:Password" "換成簽章憑證密碼" --project OAuth.AuthorizationServer
dotnet user-secrets set "Certificates:Encryption:Password" "換成加密憑證密碼" --project OAuth.AuthorizationServer
```
<span style="color: red;">這邊的密碼跟建立憑證時的密碼是一樣的</span>

3. Program.cs：改用這兩張憑證

最上面加上：
``` csharp
using System.Security.Cryptography.X509Certificates;
```

把原本這段：
``` csharp
// 開發階段用的簽章/加密憑證：之後正式環境一定要換成
// 真正保管好的簽章憑證（見設計文件第9節注意事項2、第8.1節）
options.AddDevelopmentEncryptionCertificate()
       .AddDevelopmentSigningCertificate();
```

換成：
``` csharp
// 固定的簽章/加密憑證（設計文件第9.1、10節）：路徑由設定指定、密碼放在 user-secrets（正式環境改由金鑰管理機制提供）。
// 簽章憑證：簽 JWT（RS256），公鑰發布在 JWKS；加密憑證：加密 Authorization Code、refresh_token
options.AddSigningCertificate(LoadCertificate(builder.Configuration, "Certificates:Signing"))
       .AddEncryptionCertificate(LoadCertificate(builder.Configuration, "Certificates:Encryption"));
```

在檔案最下面、SeedDataAsync 的後面新增：
``` csharp
static X509Certificate2 LoadCertificate(IConfiguration configuration, string section)
{
    var path = configuration[$"{section}:Path"]
        ?? throw new InvalidOperationException($"找不到設定 '{section}:Path'");
    var password = configuration[$"{section}:Password"]
        ?? throw new InvalidOperationException($"找不到設定 '{section}:Password'");

    return X509CertificateLoader.LoadPkcs12FromFile(path, password);
}
```

幾個重點說明：

- 為什麼 KeyUsage 要分開設：憑證的 KeyUsage 規定這把金鑰「可以做什麼」。OpenIddict 會檢查：簽章憑證必須有 DigitalSignature，加密憑證必須有 KeyEncipherment，不符合就啟動失敗。
- 為什麼產生後要從憑證存放區刪掉：New-SelfSignedCertificate 一定要先把憑證建在 Windows 憑證存放區裡，匯出成 .pfx 後就不需要了。刪掉可以確保程式真的是從檔案讀取，不是不小心從存放區讀到。
- .pfx 為什麼要放在 repo 外面：它包含私鑰。私鑰外洩的話，任何人都能簽出「看起來是我們發的」JWT。放在 repo 外面，就不會被 commit 進 git。
- X509CertificateLoader.LoadPkcs12FromFile：.NET 9 之後建議用的載入方式。舊的 new X509Certificate2(path, password) 已經被標記為過時。
- 有效期 2 年：到期前要輪替。設計文件第 9.1 節的重疊式輪替（先發布新公鑰，再切換簽發用的私鑰），之後要用時再實作。

⚠️ 換憑證之後，舊的 token 全部失效
- 舊的 access_token：用舊私鑰簽的，JWKS 裡已經沒有對應的公鑰。
- 舊的 Authorization Code、refresh_token：用舊憑證加密的，新憑證解不開。

開發環境沒關係，重跑一次流程就好。但這也說明了為什麼正式環境一定要做重疊式輪替。

測試時要注意：只動了 Authorization Server，MockPnp 不用改。

1. 重啟 Authorization Server，開啟 https://localhost:7280/.well-known/jwks，記下 kid。
2. 再重啟一次，再看一次 kid。
3. 跑一次完整流程，加上 refresh。

請執行後確認：
1. dotnet build 正常編譯，Server 正常啟動，沒有憑證相關的例外。
2. 第 1 步的 kid 和你之前看到的不同，代表換成了新的簽章憑證。
3. 第 2 步重啟後 kid 沒有變，代表憑證是固定的。
4. 第 3 步完整流程和 refresh 都正常。access_token Header 裡的 kid 和 JWKS 一致。

.well-known/jwks裡的kid
{% asset_image kid_check_1.jpg kid_check_1 %}

acess_token解碼出的jwt header。kid跟.well-known/jwks裡的kid一樣。
{% asset_image kid_check_2.jpg kid_check_2 %}

refresh_token解碼出的jwt header。kid跟.well-known/jwks裡的kid也是一樣。
{% asset_image kid_check_3.jpg kid_check_3 %}


**到這個步驟為止：改用我們自己產生、自己管理的固定憑證，取代 OpenIddict 自動產生的開發憑證。**

{% asset_image 憑證比較.jpg 憑證比較 %}

另外說明一點：<span style="color: red;">JWT 簽章用自簽憑證是正常的，正式環境也不需要向 CA 申請。</span>Resource Server 是靠 JWKS 公鑰驗簽，信任來自「JWKS 是從我們的 Authorization Server 拿的」，不是靠 CA 憑證鏈。這和網站 HTTPS 用的憑證不一樣，HTTPS 憑證才需要 CA 簽發。

