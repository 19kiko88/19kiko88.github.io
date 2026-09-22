---
title: 用IAP + Google Auth Platform 進行身分驗證
date: 2026-09-22 15:34:39
categories:
  - Google Cloud
tags:
  - IAP
  - Google Auth Platform
---

這篇記錄我把一個 ASP.NET Core 寫的小型控制面板（用來手動開關另一個 Cloud Run 服務）串上 Google 的 Identity-Aware Proxy（IAP），只讓自己的 Google 帳號能登入使用的完整過程——中間因為 Google 剛好在這段時間淘汰了舊版 IAP 設定 API，踩了不少坑，這篇把每個卡關的地方跟解法都記下來。

## 目錄

1. [背景與目標](#1-背景與目標)
2. [整體架構](#2-整體架構)
3. [完整設定流程](#3-完整設定流程)
4. [.NET 程式端要做的事（結論：幾乎不用做）](#4-net-程式端要做的事結論幾乎不用做)
5. [踩坑記錄](#5-踩坑記錄)
6. [注意事項總結](#6-注意事項總結)
7. [附錄：關鍵指令速查表](#7-附錄關鍵指令速查表)
<span style="color:red;">文章內容透過ai整理產出</span>
<!--more-->
---

## 1. 背景與目標

主服務是一個跑 Playwright 自動化訂位的 Cloud Run 服務，因為背景常駐運作的計費模式（CPU 一律配置）不便宜，所以另外做了一個小型「控制面板」（獨立的 Cloud Run 服務），可以手動開/關主服務、選擇要用的 CPU/記憶體規格。

這個控制面板本身能直接改動一個會計費的雲端資源，**絕對不能公開給任何人存取**，所以選用 Google Cloud 的 **Identity-Aware Proxy（IAP）** 來保護它——好處是完全不用在應用程式裡寫任何登入/驗證邏輯，IAP 會在網路層擋在應用程式前面，只有通過驗證的人，請求才會被放行到後面的程式。

## 2. 整體架構

```
瀏覽器
  │  1. 打開控制面板網址
  ▼
IAP（Google 管理，擋在服務前面）
  │  2. 沒登入 → 導去 Google 登入頁
  │  3. 登入後檢查這個身分有沒有被授權
  ▼
Cloud Run 服務（.NET 應用程式）
  │  4. 完全不知道剛剛發生了什麼驗證流程，
  │     直接處理請求（IAP 已經幫你把關）
  ▼
呼叫 Cloud Run Admin API 改動另一個服務的設定
```

關鍵認知：**認證（你是誰）跟授權（你能不能通過）是兩件分開的事**。任何人都可以在 Google 登入頁輸入自己的帳密成功登入（這是認證），但 IAP 還會另外檢查這個身分有沒有被加進白名單（這是授權）——沒有被加進去的帳號，登入成功後一樣會被擋下來，不會看到後面的頁面。

---

## 3. 完整設定流程

### OAuth 跟 IAP 各自的用途，以及兩者的關係

在動手設定之前，先搞懂這兩個名詞各自在做什麼，之後每個步驟才不會覺得「不知道在設定什麼」：

- **OAuth（Google Auth Platform）的用途**：OAuth 是 Google 用來**驗證身分**的協定本身——使用者用 Google 帳號登入、Google 確認密碼/身分無誤後，發一組憑證證明「這個人真的是他自稱的那個帳號」。「Google Auth Platform」（品牌、目標對象、用戶端）就是設定「誰可以發起這個登入流程」：品牌決定登入頁上顯示什麼 App 資訊；目標對象決定哪些帳號可以嘗試登入（測試使用者名單）；用戶端（Client ID/Secret）則是這個特定應用程式在 Google 那邊的身分證，讓 Google 知道是誰在請求登入、登入完要把使用者送回哪裡。**OAuth 本身不管「這個資源該不該給這個人看」，它只負責回答「這個人是誰」。**

- **IAP 的用途**：IAP 是 Google Cloud 的**存取控制關卡**，擋在你的 Cloud Run 服務前面，決定「這個資源到底該不該讓這個請求進來」。它自己不會發明一套登入機制，而是**借用**上面設定好的 OAuth 流程去驗證使用者身分，驗證完之後，再多做一件 OAuth 不會做的事：檢查這個身分有沒有在白名單（IAM）裡，決定要不要把請求真正放行到後面的應用程式。

- **兩者的關係**：可以理解成「OAuth 負責認證，IAP 負責把認證結果拿來做授權判斷、並實際把關」。IAP 需要吃到一組 OAuth 用戶端（Client ID/Secret）才能運作，因為它要靠這組用戶端去跟 Google 的 OAuth 伺服器對話、觸發登入畫面；沒有先設定好 OAuth 那一側，IAP 就沒有可以用來驗證身分的管道，這也是為什麼下面的設定順序要先做 OAuth、再啟用 IAP。

### 3.0 流程總覽

實際踩坑後，建議的正確順序是這樣（跟一開始直覺想的「先開 IAP 再設定 OAuth」剛好反過來）：

1. **OAuth（Google Auth Platform）先設好**：品牌 → 目標對象（含加測試使用者）→ 用戶端（建立 + 補上重新導向 URI）
2. **啟用 IAP**：在 Cloud Run 服務上打開 IAP 開關
3. **把用戶端交給 IAP**：回到 IAP 頁面，把步驟 1 建立的 Client ID / Secret 填進**這個資源**的 IAP 設定——這步最容易被忽略，沒做的話 IAP 不知道要用哪組 OAuth 用戶端，會出現 `Empty Google Account OAuth client ID(s)/secret(s)` 錯誤（見 5.2）
4. **IAP 白名單設定**：`gcloud iap web add-iam-policy-binding`，決定登入成功後誰真的能通過

我們當時是先做了步驟 2（啟用 IAP），才回頭補步驟 1（OAuth 設定），這個順序顛倒導致 IAP 沒能正確綁定用戶端，後來要多繞一次「關閉再重新啟用 IAP」（`--no-iap` 再 `--iap`）才讓它重新嘗試綁定，詳見 5.5。**如果照 1→2→3→4 的順序做，應該可以省掉這個回頭步驟。**

### 3.1 設定 Google Auth Platform（OAuth 同意畫面）

Google 最近把整個 OAuth 同意畫面的設定畫面重新設計、重新命名（舊版叫「OAuth consent screen」，新版整合進「Google Auth Platform」），進入點：

```
https://console.cloud.google.com/auth/overview?project=<PROJECT_ID>
```

第一次進去會顯示「尚未設定 Google 驗證平台」，點下去開始設定：

1. **應用程式資訊**：App 名稱隨便取、使用者支援信箱填自己的
2. **目標對象**：個人帳號只會看到「外部」這個選項（沒有「內部」，那是 Workspace 組織才有的）
3. **聯絡資訊**：填自己的信箱
4. 完成後，進到「**目標對象**」分頁，把自己的 Google 帳號加進「**測試使用者**」清單——這一步決定了只有加進來的帳號才能通過（維持在「測試中」狀態即可，不需要送 Google 審核）

### 3.2 手動建立 OAuth 用戶端

到「**用戶端**」分頁，點「建立用戶端」：

- **應用程式類型**：網頁應用程式
- **名稱**：隨便取，方便識別即可（例如 `IAP <服務名稱>`）
- 先不填重新導向 URI，直接建立，拿到一組 **Client ID**（長得像 `123456789012-abcdefg.apps.googleusercontent.com`）
{% asset_image create_oauth.jpg create_oauth  %}

建立完後，回去編輯這個用戶端，在「**已授權的重新導向 URI**」（不是「已授權的 JavaScript 來源」，這兩個是不同欄位，功能也不同——JavaScript 來源不接受路徑，重新導向 URI 才接受）填入：

```
https://iap.googleapis.com/v1/oauth/clientIds/<完整的CLIENT_ID>:handleRedirect
```

把 `<完整的CLIENT_ID>` 換成剛剛拿到的**整組** Client ID（不是取一部分），存檔。
{% asset_image create_oauth2.jpg create_oauth2  %}

### 3.3 在 Cloud Run 服務上啟用 IAP

```bash
gcloud run services update <SERVICE_NAME> --region=<REGION> --iap
```

執行後如果專案不屬於 Google Workspace 組織（也就是用個人 Gmail 帳號在跑），會看到這個警告：

```
[Warning] Deploying services with IAP enabled in a project outside of an
Organization and may require initial setup via the Cloud Console.
```

因為 3.1、3.2 都已經先設定好了，這裡的警告可以放心忽略，繼續下一步。

### 3.4 把 OAuth 用戶端交給 IAP

回到 IAP 頁面：

```
https://console.cloud.google.com/security/iap?project=<PROJECT_ID>
```

點進對應的服務資源，把 3.2 建立的 Client ID / Client Secret 填進去。

> 如果順序不小心做反了（先啟用 IAP、後補 OAuth 設定），IAP 可能沒能正確綁定用戶端，需要多一步「關閉再重新啟用 IAP」讓它重新嘗試綁定：
> ```bash
> gcloud run services update <SERVICE_NAME> --region=<REGION> --no-iap
> gcloud run services update <SERVICE_NAME> --region=<REGION> --iap
> ```

### 3.5 把自己的帳號加進 IAP 白名單

這一步才是真正的「授權」，決定誰能通過：

```bash
gcloud iap web add-iam-policy-binding \
  --resource-type=cloud-run --service=<SERVICE_NAME> --region=<REGION> \
  --member="user:<你的Email>" \
  --role="roles/iap.httpsResourceAccessor"
```

（也可以在 IAP 網頁上點「新增主體」達到同樣效果。）
{% asset_image ipa_allow_user.jpg ipa_allow_user  %}

### 3.6 實際測試

用瀏覽器打開服務網址，應該會被導向 Google 登入頁，登入自己的帳號後，就能看到後面真正的頁面。

---

## 4. .NET 程式端要做的事（結論：幾乎不用做）

這是最值得記錄的一點：**IAP 是網路層的保護機制，應用程式完全不需要寫任何登入/驗證相關的程式碼**。整個 ASP.NET Core Minimal API 專案裡，沒有一行跟 IAP、OAuth、登入有關的程式——IAP 會在請求送到 Cloud Run 容器之前就完成驗證，容器裡的程式收到的，永遠是已經通過驗證的請求。

如果應用程式想知道「現在是誰在操作」，IAP 會在請求標頭裡附加 `X-Goog-IAP-JWT-Assertion`（一組已驗證過身分的 JWT），程式可以選擇性地解析這個標頭拿到使用者身分；但如果只是要「擋掉沒有權限的人」，完全不需要做這件事，IAP 本身已經處理好了。

---

## 5. 踩坑記錄

### 5.1 舊版 IAP 設定指令已被淘汰

```bash
gcloud iap oauth-brands list
```

會看到：

```
WARNING: This command is deprecated and will be non-functional after the
IAP OAuth Admin APIs are turned down. Jan 19, 2026: ... March 19, 2026: ...
```

`oauth-brands`、`oauth-clients` 這兩組指令已經停用，現在管理「誰能通過 IAP」要用：

```bash
gcloud iap web add-iam-policy-binding ...
gcloud iap web get-iam-policy ...
```

而 OAuth 同意畫面/用戶端本身，改到 Google Auth Platform 的網頁介面設定，沒有對應的 gcloud 指令可以完全自動化（至少目前沒有）。

### 5.2 502 錯誤但看不到原因

啟用 IAP 後直接打網址，只看到:

```
502 Bad Gateway
```

**沒有任何進一步資訊**。PowerShell 7 的 `Invoke-WebRequest` 遇到非 2xx 狀態碼會直接丟例外，預設的例外處理拿不到回應本體內容（`$_.Exception.Response` 是新版 `HttpResponseMessage`，沒有舊版 `GetResponseStream()` 方法）。真正有效的做法：

```powershell
$resp = Invoke-WebRequest -Uri $url -UseBasicParsing -SkipHttpErrorCheck
$resp.Content
```

`-SkipHttpErrorCheck` 讓它無論狀態碼為何都直接回傳回應物件，這樣才看到實際錯誤內容：

```
Empty Google Account OAuth client ID(s)/secret(s).
```

這句話才是真正的線索，指向「還沒建立 OAuth 用戶端」這個根本原因。

### 5.3 重新導向 URI 填錯欄位

第一次嘗試時把 IAP 要求的重新導向 URI 填進了「**已授權的 JavaScript 來源**」，出現：

```
來源無效：URI 不得包含路徑或以「/」結尾。
```

因為 JavaScript 來源欄位規定只能填網域本身，不能有路徑；IAP 要的那個帶路徑的完整網址，要填在「**已授權的重新導向 URI**」這個不同的欄位。這兩個欄位在畫面上通常排在一起，很容易搞混。

### 5.4 手動測試 IAP 保護的網址

一旦 IAP 設定好，原本用來測試的 `gcloud auth print-identity-token` 產生的通用身分權杖不能直接用了：

```
Invalid IAP credentials: Invalid bearer token. Invalid JWT audience.
```

因為 IAP 要求權杖的 `audience` 必須跟這個資源綁定的 OAuth Client ID 一致，通用的身分權杖 audience 不對，會直接被拒絕。這種情況下，最直接的驗證方式就是**用真正的瀏覽器手動登入測試**，不強求整個流程都能腳本化驗證。

### 5.5 順序很重要：先設品牌，再啟用 IAP

一開始的操作順序是：先跑 `--iap`，之後才去設定 Google Auth Platform 的品牌/目標對象。這個順序造成 IAP 沒能正確建立/綁定 OAuth 用戶端，即使後來把品牌設定補上，也要**重新關閉再開啟一次 IAP**（`--no-iap` 再 `--iap`）才會重新嘗試綁定。如果重來一次，建議順序改成：先把 Google Auth Platform（品牌、目標對象、測試使用者）都設好，再啟用 IAP，可能可以省掉這個回頭重跑的步驟。

---

## 6. 注意事項總結

- **個人 Gmail 專案 ≠ Workspace 組織專案**：IAP 在組織帳號下可以自動處理掉的事（OAuth 用戶端自動建立），在個人帳號下都要手動做一次，這是最大的差異來源。
- **認證跟授權分開設定**：Google Auth Platform（品牌/測試使用者）決定「誰可以嘗試登入」；`gcloud iap web add-iam-policy-binding`（`roles/iap.httpsResourceAccessor`）才決定「登入後誰真的能通過」。兩邊都要設，缺一個都不會通。
- **應用程式端不用寫任何驗證程式碼**——如果你發現自己在應用程式裡寫 OAuth callback、驗證 JWT 之類的邏輯來源自「因為要串 IAP」，那通常代表方向錯了，這些事應該完全交給 IAP 處理。
- **診斷問題時，先看回應本體內容，不要只看狀態碼**——單看 502/403 這種狀態碼幾乎看不出真正原因，錯誤訊息通常藏在回應本體（body）裡。
- **腳本化測試在 IAP 保護的資源上會受限**——通用身分權杖的 audience 跟 IAP 要求的不一致時會直接被拒絕，這種情況接受用真人瀏覽器測試，不要硬要全自動化。

---

## 7. 附錄：關鍵指令速查表

```bash
# 在 Cloud Run 服務上啟用/停用 IAP
gcloud run services update <SERVICE> --region=<REGION> --iap
gcloud run services update <SERVICE> --region=<REGION> --no-iap

# 管理誰能通過 IAP（現行、未淘汰的指令）
gcloud iap web add-iam-policy-binding \
  --resource-type=cloud-run --service=<SERVICE> --region=<REGION> \
  --member="user:<EMAIL>" --role="roles/iap.httpsResourceAccessor"

gcloud iap web get-iam-policy \
  --resource-type=cloud-run --service=<SERVICE> --region=<REGION>

# 診斷 IAP 保護網址的實際錯誤內容（PowerShell）
$resp = Invoke-WebRequest -Uri $url -UseBasicParsing -SkipHttpErrorCheck
$resp.StatusCode
$resp.Content

# 重新導向 URI 格式（填在 OAuth 用戶端的「已授權的重新導向 URI」）
https://iap.googleapis.com/v1/oauth/clientIds/<CLIENT_ID>:handleRedirect
```

---

*這篇記錄的是 2026 年 9 月當下的 Google Cloud Console 介面跟流程，Google 正在淘汰舊版 IAP OAuth Admin API、重新設計 Auth Platform 介面，未來畫面跟指令可能會再變動，實際操作請以當時 Console 上顯示的內容為準。*

