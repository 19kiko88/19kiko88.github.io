---
title: JWT、JWKS 與 kid 的關係
date: 2026-11-16 15:25:35
categories:
  - 後端開發
tags:
  - OAuth
  - OpenIddict
  - JWT
  - JWKS
---

> 本文件說明 Authorization Server 簽發的 access_token 如何被驗證，以及簽章憑證輪替時 `kid` 的作用。相關規格見 `OAuth2.1-MCP-AuthServer-Design.md` 第9.1、9.3節。
<!-- more -->
---

## 1. 一句話說明

Authorization Server 用**私鑰**簽 JWT，把對應的**公鑰**公開在 **JWKS**；JWT Header 裡的 **`kid`** 告訴驗證方「該拿 JWKS 裡哪一把公鑰來驗」。

## 2. 比喻：蓋了印章的公文

{% asset_image jwt_jwks_kid.jpg jwt_jwks_kid %}

---

## 3. JWT：三段組成

{% asset_image jwt.jpg jwt %}

- Header 與 Payload 只是 Base64Url 編碼，**任何人都能解開來看**，不可放機密資料。
- 沒有私鑰就算不出正確的 Signature，所以內容只要被改過一個字，簽章就對不上。

在本機解開 Header（不必把 token 貼到外部網站）：

```powershell
$jwt = '貼上 access_token'
$header = $jwt.Split('.')[0].Replace('-', '+').Replace('_', '/')
switch ($header.Length % 4) { 2 { $header += '==' } 3 { $header += '=' } }
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($header))
```

把 `Split('.')[0]` 改成 `Split('.')[1]` 即可看 Payload。

---

## 4. JWKS：公鑰清單

```json
{
  "keys": [
    { "kid": "ABC123", "kty": "RSA", "alg": "RS256", "use": "sig", "n": "...", "e": "AQAB" },
    { "kid": "XYZ789", "kty": "RSA", "alg": "RS256", "use": "sig", "n": "...", "e": "AQAB" }
  ]
}
```

- 每一筆是一把公鑰；`n`、`e` 是 RSA 公鑰的數值。
- JWKS **只有公鑰**，可以公開給任何人。
- 由 OpenIddict 依登記的簽章憑證自動產生，所有登記的簽章憑證（包含尚未生效的）都會出現在這裡。
- JWKS 的網址可從 Discovery（`/.well-known/openid-configuration`）的 `jwks_uri` 取得。

---

## 5. kid：把 JWT 和 JWKS 串起來

MCP Resource Server 收到 access_token 時的驗證步驟：

```
1. 解開 JWT 的 Header → kid = "ABC123"
2. 到 JWKS 找 kid = "ABC123" 的那一把公鑰（第一次抓下來後通常會快取）
3. 用這把公鑰驗證 Signature
   → 驗證成功：證明是我們的 Authorization Server 簽的，內容也沒被改過
   → 驗證失敗或找不到這個 kid：拒絕
4. 再檢查 Payload：exp（過期沒）、iss（誰發的）、aud（是不是發給我的）、scope（能呼叫哪些 tools）
```

**為什麼需要 `kid`**：JWKS 永遠只有一把公鑰時 `kid` 不太必要；但**輪替期間 JWKS 會同時有兩把**，驗證方必須知道「這顆 token 該用哪一把驗」，`kid` 就是用來指定的。

---

## 6. refresh_token 的 kid 不是簽章憑證

| | access_token | refresh_token |
|---|---|---|
| 格式 | JWT（3 段） | 加密過的 JWT，即 JWE（5 段） |
| 看得到的 Header 的 `kid` | **簽章憑證** | **加密憑證** |
| 誰會讀 | Resource Server（外部） | 只有 Authorization Server 自己 |

refresh_token 的結構是「先用簽章憑證簽好，再整個用加密憑證加密包起來」：

```
refresh_token（外層，看得到）
├ Header：{"alg":"RSA-OAEP", "enc":"A256CBC-HS512", "kid":"<加密憑證的kid>"}
└ 加密的內容（看不到）
   └ 裡面才是一個簽過章的 JWT，Header 的 kid = 簽章憑證
```

所以 refresh_token 外層的 `kid` 不能拿來和 access_token 的 `kid` 比較。

---

## 7. 簽章憑證的重疊式輪替

### 7.1 NotBefore 與 NotAfter

每張憑證都有有效期間：

| | 意思 |
|---|---|
| `NotBefore` | 生效日：在這個時間**之前**，憑證無效 |
| `NotAfter` | 到期日：在這個時間**之後**，憑證無效 |

```
          NotBefore                              NotAfter
              │◀───────── 有效期間 ──────────────▶│
──────────────┼──────────────────────────────────┼──────────▶ 時間
   還沒生效     │            可以使用               │   已過期
```

### 7.2 OpenIddict 選擇簽章憑證的規則

1. 還沒到 `NotBefore` 的憑證，不會被用來簽發。
2. 在已生效的憑證裡，`NotAfter` 最晚的優先。
3. 選擇在啟動時決定，因此新憑證生效後需要**重啟（或部署）**才會切換。

### 7.3 輪替步驟

1. 在舊憑證到期前，把新憑證加進設定：`NotBefore` 設在未來、`NotAfter` 比舊的晚。新公鑰**立刻出現在 JWKS**，但還不會用來簽發（**先發布，晚使用**）。
2. 新憑證生效前：不論是用 Authorization Code 還是 refresh_token 換到的 access_token，`kid` 都是舊憑證的。
3. 新憑證生效、重啟之後：兩種方式換到的新 access_token，`kid` 都變成新憑證的。
4. 重疊期間，JWKS 同時有新舊兩把公鑰；**同一時間只有一張憑證負責簽發**——新憑證簽發新 token，舊憑證繼續驗證之前發出的 token，使用者不會被強制登出。
5. 等舊憑證簽的 token 都過期（refresh_token 最長 12 小時）後，把舊憑證從設定移除。

`NotBefore` 至少要大於「Resource Server 快取 JWKS 的時間 + access_token 效期（30 分鐘）」，正式環境建議設幾天。

### 7.4 各階段的 kid

{% asset_image kid_compare.jpg kid_compare %}

**為什麼要「先發布，晚使用」**：如果新憑證一加入就立刻簽發，Resource Server 手上快取的 JWKS 還沒有 `XYZ789`，會遇到不認得的 `kid` 而驗證失敗。

### 7.5 加密憑證的輪替

加密憑證只有 Authorization Server 自己使用，公鑰不會發布到 JWKS，沒有外部快取的問題：

- 加入新的加密憑證時**不需設 `NotBefore`**，`NotAfter` 比舊的晚即可，重啟後立刻改用新的加密。
- 舊的加密憑證保留到所有用它加密的 Authorization Code、refresh_token 過期（最長 12 小時）後再移除。

### 7.6 設定方式（開發環境）

`appsettings.Development.json`（路徑）：

```json
"Certificates": {
  "Signing": [
    { "Path": "D:\\Homer\\Secrets\\oauth-signing.pfx" },
    { "Path": "D:\\Homer\\Secrets\\oauth-signing-2.pfx" }
  ],
  "Encryption": [
    { "Path": "D:\\Homer\\Secrets\\oauth-encryption.pfx" }
  ]
}
```

user-secrets（密碼，索引與上面的陣列對應）：

```
Certificates:Signing:0:Password
Certificates:Signing:1:Password
Certificates:Encryption:0:Password
```

移除舊憑證時，陣列與 user-secrets 的索引要一起調整（剩下的憑證要從 `0` 開始）。

