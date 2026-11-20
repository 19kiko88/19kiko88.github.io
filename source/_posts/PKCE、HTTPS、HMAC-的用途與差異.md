---
title: PKCE、HTTPS、HMAC 的用途與差異
date: 2026-11-20 10:08:39
categories:
  - 後端開發
tags:
  - OAuth
  - OpenIddict
---
<!-- more -->

> 本文件整理本專案用到的三種安全機制：各自保護什麼、在專案中的哪裡使用、彼此如何分工。相關規格見 `OAuth2.1-MCP-AuthServer-Design.md` 第4.2、4.4節，HMAC 與 PNP 的實作細節見 `Handoff-ResourceServer.md` 第8.3節。

---

## 1. 一句話說明

| 機制 | 一句話 |
|---|---|
| **PKCE** | 讓被偷走的 Authorization Code 無法換到 access_token |
| **HTTPS** | 加密連線（第三方看不到）、防止傳輸途中被竄改、確認 **Server** 的身分 |
| **HMAC** | 確認**呼叫者**持有 Secret，而且請求內容沒有被竄改 |

HTTPS 與 HMAC 都能防竄改，關鍵差別在「**確認誰的身分**」：HTTPS 確認 **Server**，HMAC 確認 **Client（呼叫者）**。

---

## 2. PKCE（Proof Key for Code Exchange）

### 2.1 解決的問題

Authorization Code 會透過瀏覽器導轉送回 MCP Client（`redirect_uri?code=...`），途中可能被攔截（瀏覽器歷史紀錄、惡意程式、log）。沒有 PKCE 時，偷到 code 的人可以直接拿去 `/connect/token` 換 access_token。

PKCE **無法阻止 code 被偷走**，而是讓**偷走的 code 換不到 token**。

### 2.2 運作方式

```
1. MCP Client 產生只有自己知道的隨機字串 code_verifier
2. 算出 code_challenge = Base64Url(SHA256(code_verifier))
3. /connect/authorize 時送出 code_challenge（會出現在網址上，可能被看到）
4. Authorization Server 把 code_challenge 和簽發的 Authorization Code 綁在一起
5. /connect/token 時，MCP Client 送出 code 與原始的 code_verifier
6. Authorization Server 計算 SHA256(code_verifier)，與當初的 code_challenge 比對，相同才簽發 token
```

偷到 code（甚至看到 `code_challenge`）的人，手上沒有 `code_verifier`，而 SHA256 無法反推，所以換不到 token。

### 2.3 為什麼只允許 S256，不允許 plain

| 方法 | `code_challenge` 的算法 | 被看到 `code_challenge` 時 |
|---|---|---|
| `S256` | `SHA256(code_verifier)` | 無法反推 `code_verifier`，安全 |
| `plain` | 直接等於 `code_verifier` | `code_challenge` 本身就是通關密碼，PKCE 失去作用 |

### 2.4 在本專案中的位置

- Authorization Server 強制所有 client 使用 PKCE，且**只允許 `S256`**（設計文件第10節第4點）。
- 第二次經過 `/connect/authorize` 時比對參數，防止 resume 網址中的 `code_challenge` 被替換（設計文件第4.1節）。

---

## 3. HTTPS

### 3.1 保護的範圍

| 保護 | 說明 |
|---|---|
| **加密** | 連線雙向加密，第三方看不到傳輸內容（token、帳號、資料） |
| **防竄改** | **雙向**保護，Client 送出的請求與 Server 回傳的回應，傳輸途中都無法被修改 |
| **確認 Server 身分** | Client 驗證 Server 的憑證，確認連到的是真的 Server，而不是假冒者 |

### 3.2 沒有保護的部分

一般的 HTTPS **不驗證 Client 是誰**。任何人都能和 PNP 建立 HTTPS 連線並送出請求，HTTPS 本身無法分辨請求是來自 Authorization Server、MCP Server，還是攻擊者。這正是需要 HMAC 的原因。

（雙向驗證的 mTLS 可以驗證 Client，設計文件第3.1節將它列為確認為可信內網後「可疊加」的防護層，目前未採用。）

### 3.3 在本專案中的位置

所有連線都要求 HTTPS：MCP Client ↔ Authorization Server、MCP Client ↔ MCP Server、Authorization Server → PNP、MCP Server → PNP。

---

## 4. HMAC

### 4.1 保護的範圍

| 保護 | 說明 |
|---|---|
| **確認呼叫者身分** | 只有持有 Secret 的一方能算出正確的簽章 |
| **防竄改** | 內容被改過一個字元，簽章就對不上 |
| **防重送** | 搭配時間戳（±5 秒），被攔截的請求過了時限就無法再使用 |
| ❌ 不加密 | HMAC 只簽章、不加密，內容的保密仍靠 HTTPS |

### 4.2 運作方式

**Secret 從頭到尾都不會被傳送，也不會被拿來比對**；比對的是雙方各自算出的簽章。

```
送出端（Authorization Server / MCP Server）                 接收端（PNP）
───────────────────────────────────                    ─────────────────
Secret（自己保管，不送出）                                  Secret（自己保管，同一個值）
        │
        ▼
簽章 = HMAC-SHA256(Secret, "時間戳\n方法\n路徑\nbody雜湊")
        │
        └── X-Timestamp、X-Signature、body ──────────▶  用自己的 Secret 對收到的內容重算一次
                                                          兩個簽章相同？ → 通過
                                                          時間差 ±5 秒內？ → 通過
```

- **HMAC-SHA256**：使用 SHA256 雜湊函數的 HMAC。在 JWT 中，同一個演算法的簡寫是 `HS256`。
- 從簽章無法反推 Secret；沒有 Secret 就算不出正確的簽章。

### 4.3 驗章成功的條件

1. **Secret 相同**：雙方設定同一個值（設定 key 名稱可以不同，例如 `IdentityProviders:Pnp:HmacSecret` 與 `Sso:HmacSecret`）。
2. **計算方式完全相同**：

| 項目 | 必須一致的內容 |
|---|---|
| 簽章內容的組法 | `時間戳\n方法\n路徑\nbody雜湊`：順序與換行符號（`\n`）都相同 |
| 時間戳 | `X-Timestamp` header 的值，Unix 秒數 |
| 方法 | 大寫，例如 `POST` |
| 路徑 | 只取路徑，不含主機與 query string；**有 IIS 虛擬目錄時要包含它** |
| body 雜湊 | 對**原始 body 文字**算 SHA256，轉成**小寫** 16 進位；不可先反序列化再序列化 |
| 文字編碼 | UTF-8 |
| 演算法 | HMAC-SHA256 |
| 簽章表示 | Base64，放在 `X-Signature` header |

3. **時間同步**：接收端檢查 `X-Timestamp` 與自己的時間相差不超過 ±5 秒，所以主機時鐘須對齊（公司 NTP）。

任一項不同，合法的請求也會得到 401。

### 4.4 在本專案中的位置

| 路徑 | 用途 | 密鑰 |
|---|---|---|
| Authorization Server → PNP | redeem、token-status（目前由 MockPnp 模擬） | Authorization Server ↔ PNP 這一組 |
| MCP Server → PNP | UCS 資料的內部 API、Native Capability API | MCP Server ↔ PNP 這一組（**與上面不同**） |

兩條路徑使用不同的密鑰：一組外洩只影響一條路徑。MCP Server → PNP 的密鑰尤其重要，外洩時持有者能以任何使用者身分呼叫 Native Capability API（`Handoff-ResourceServer.md` 第2.2節）。

---

## 5. 三者如何搭配

| 連線 | HTTPS | PKCE | HMAC |
|---|---|---|---|
| MCP Client → Authorization Server（`/connect/authorize`、`/connect/token`） | ✅ | ✅ | — |
| MCP Client → MCP Server（帶 access_token 呼叫工具） | ✅ | — | —（以 JWT 驗證） |
| Authorization Server → PNP（redeem、token-status） | ✅ | — | ✅ |
| MCP Server → PNP（內部 API） | ✅ | — | ✅ |

- **PKCE** 只用在 OAuth 的授權碼流程。
- **HTTPS** 每一條連線都要。
- **HMAC** 用在伺服器對伺服器的內部 API：這些 API 沒有使用者登入狀態，靠 HMAC 確認呼叫者是誰。
- MCP Client → MCP Server 不用 HMAC，而是用 Authorization Server 簽發的 **JWT**（access_token）確認使用者與權限（見 `JWT-JWKS-kid.md`）。
