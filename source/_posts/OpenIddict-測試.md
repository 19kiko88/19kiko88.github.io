---
title: OpenIddict 測試
date: 2026-10-27 15:38:25
categories:
  - 後端開發
tags:
  - OAuth
  - OpenIddict
---
##### OpenIddict 測試
<!-- more -->
---
## 1. 產生code_verifier跟code_challenge

MCP官方的Authorization規範（建立在OAuth2.1之上）明文要求所有走Authorization Code flow的MCP client都必須實作PKCE，所以任何符合規範的MCP client（包括Claude Desktop），理論上都會自動做這整套，不需要我們的Authorization Server額外提醒或處理。
PKCE的code_verifier跟code_challenge是由Client端自己產生的——正式運行時由Claude Desktop自動產生；我們現在因為還沒有真的MCP Client，所以用PowerShell手動扮演Client的角色，自己產生這組值，再送給Authorization Server驗證。Authorization Server本身不會、也不需要產生這兩個值。

在PowerShell裡跑這段（我把輸出導向存成一個文字檔，這樣就算終端機關掉也不會不見）：

``` bash
$bytes = New-Object byte[] 32
$rng = New-Object System.Security.Cryptography.RNGCryptoServiceProvider
$rng.GetBytes($bytes)
$codeVerifier = [Convert]::ToBase64String($bytes).TrimEnd('=').Replace('+','-').Replace('/','_')

$sha256 = [System.Security.Cryptography.SHA256]::Create()
$hash = $sha256.ComputeHash([System.Text.Encoding]::ASCII.GetBytes($codeVerifier))
$codeChallenge = [Convert]::ToBase64String($hash).TrimEnd('=').Replace('+','-').Replace('/','_')

"code_verifier  = $codeVerifier" | Out-File -FilePath "$env:USERPROFILE\Desktop\pkce_test.txt" -Encoding utf8
"code_challenge = $codeChallenge" | Out-File -FilePath "$env:USERPROFILE\Desktop\pkce_test.txt" -Encoding utf8 -Append

Write-Host "code_verifier  = $codeVerifier"
Write-Host "code_challenge = $codeChallenge"
Write-Host "---"
Write-Host "已存到桌面的 pkce_test.txt，不要關掉這個檔案"
```

你也可以把上面shell存成.ps1檔案，再透過PowerShell執行，最後會把code_verifier跟code_challenge存於文字檔裡面。
``` bash
powershell -ExecutionPolicy Bypass -File PKCE.ps1
```
#
#
#
---
## 2. 呼叫驗證伺服器的api [authorize]
模擬MCP Client第一次呼叫驗證伺服器依 client_id 查詢 Identity Provider = PNP Bridge，並取得nonce，再導頁到bridge驗證頁面。
{% asset_image authorize-1.jpg authorize-1 %}

- 說明
redirect_uri：MCP Client(Claude Desktop)的網址
#
#
#
---
## 3. 呼叫驗證伺服器的api [bridge-callback]
模擬PNP的Bridge驗證登入成功後，產生BridgeCode，回呼驗證伺服器的api [bridge-callback]；[bridge-callback]收到BridgeCode後，呼叫[RedeemAsync]換取使用者相關資訊（userName/siteCode/tokenId/ticketExpiration），寫入暫存資料後，再導回api [authorize]，並執行[CompleteAuthorizationAsync]完成SignIn，<span style="color: red;">取得authorization code。</span>
{% asset_image bridge-callback.jpg bridge-callback %}

- 說明
nonce：由上一個步驟302轉址的QueryString取得
#
#
#
---
## 4. 呼叫驗證伺服器的api [token]
模擬MCP Client從用authorization code從驗證伺服器取得access token。
{% asset_image token.jpg token %}

acess_token為JWT，可以用工具解析payload內容。
{% asset_image jwt_payload.jpg jwt_payload %}

- 說明
code：由上一個步驟302轉址的QueryString取得
code_verifier：步驟1.產生的code_verifier