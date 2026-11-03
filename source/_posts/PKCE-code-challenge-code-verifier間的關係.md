---
title: PKCE. code_challenge. code_verifier間的關係
date: 2026-11-02 10:29:17
categories:
  - 後端開發
tags:
  - OAuth
  - OpenIddict
---
##### 
PKCE（Proof Key for Code Exchange）是 OAuth 2.0 Authorization Code Flow 的安全強化機制。
code_verifier 和 code_challenge 是一組配對的字串，目的在於：
確保取得 Authorization Code 的 Client，才有能力用這組 Code 換取 Access Token。
<!-- more -->
兩者最大的差別是：
- code_verifier：原始驗證字串，由 Client 自己產生並保存。
- code_challenge：由 code_verifier 經過 SHA-256 雜湊後產生，傳給 OAuth Server。
#
#
#
---
## 1. 兩者的差異
項目	code_verifier	code_challenge
中文	驗證原始碼	驗證挑戰碼
產生者	Client	Client
產生方式	隨機字串	SHA-256 雜湊
傳送對象	Token Endpoint	Authorization Endpoint
用途	最後提供給 Server 驗證	讓 Server 預先保存

計算方式：
``` text
code_challenge = BASE64URL(SHA256(code_verifier))
```
注意：實際 OAuth PKCE 通常使用 S256，不是直接使用一般 Base64。
#
#
#
---
## 2. 實際運作流程

### 2.1
**Client 產生兩組資料**
- 隨機產生 code_verifier
- 計算出 code_challenge

Client 保存
code_verifier = 原始隨機字串

送給 OAuth Server
code_challenge = SHA256(verifier)

### 2.2
**Client 呼叫 Authorization Endpoint**
將 code_challenge 傳給 OAuth Server。

### 2.3
**OAuth Server 發出 Authorization Code**
Server 將這次授權流程的 code_challenge 與 Authorization Code 綁定。

### 2.4
**Client 呼叫 Token Endpoint**
用 Authorization Code 換取 Access Token，並附上原始的 code_verifier。

### 2.5
**OAuth Server 驗證**
Server 自己對收到的 code_verifier 執行 SHA-256，再與先前保存的 code_challenge 比較。
- 相同：通過 PKCE 驗證。
- 不同：拒絕核發 Access Token。
#
#
#
---
## 3. 為什麼不直接傳 code_verifier？
假設攻擊者攔截了 Authorization Code：
- 攻擊者知道 Authorization Code。
- 但不知道 Client 保存的 code_verifier。
- 因此無法通過 Token Endpoint 的 PKCE 驗證。
- OAuth Server 不會核發 Access Token。
這就是 PKCE 的安全價值。
#
#
#
---
## 4. 套用到你的 OpenIddict MCP 架構
你可以這樣理解：
- MCP Client：產生 code_verifier、code_challenge。
- OpenIddict OAuth Server：保存並驗證 code_challenge。
- Token Endpoint：收到 code_verifier 後進行比對。
- 驗證成功：才繼續核發 Access Token。
#
#
#
---
另外要特別注意：PKCE 不等於 Client Authentication，也不能取代 Client Secret。 它是額外的一層安全保護。
OAuth 2.0+1

官方規範：RFC 7636 – PKCE

最簡單的記法：
code_verifier 是原始答案，code_challenge 是答案經過雜湊後的結果。Client 先交出結果，之後再交出原始答案讓 Server 驗證。