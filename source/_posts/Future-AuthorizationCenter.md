---
title: Future-AuthorizationCenter
date: 2026-11-20 18:33:57
password: 7533967
categories:
  - 後端開發
tags:
  - OAuth
  - OpenIddict
---
# 未來規劃：以 OAuth 作為集中授權中心

> 狀態：**未來規劃（評估階段，尚未定案、尚未排入開發）**。本文件獨立於目前的 Authorization Server／MCP Resource Server 規格與交接文件；其他文件不因本規劃而修改，待定案後再行調整。
> 建立日期：2026-11-20

---

## 1. 需求

| 項目 | 內容 |
|---|---|
| 身分驗證（你是誰） | **保留 PNP 原本的登入驗證頁**（登入驗證功能非本部門維護）。通過登入驗證後，取得使用者的 **Windows AD 帳號** |
| 授權（你能做什麼） | AD 使用者後續的權限（authority）處理，**改由 OAuth（Authorization Server）接手**，與原本的 PNP 權限做對應（mapping） |
| 外部人員 | `ucs\` 帳號＋OTP 的外點維修人員**不受影響**，維持 PNP 原本的權限控管 |
| 長期目標 | OAuth 不只用於 PNP，日後也用於**其他內部系統**的權限控管，達到**透過 OAuth 進行單一授權管理** |

換言之，Authorization Server 的角色由目前的「**身分代理**」（確認身分、簽發 token），擴充為「**授權中心**」（記錄每個人在每個系統能做什麼）。

---

## 2. 現況（2026-11-20 於 PNP repo 確認）

| 項目 | 現況 |
|---|---|
| AD 登入 | `AccountController.Login` 已透過 LDAP 驗證 AD 帳密（`LdapAuthentication.IsAuthenticated(domain, account, passWord)`）。另有非 AD 的外點人員登入：`ucs\` 帳號＋OTP |
| 權限資料 | `Authority` 表，以 `Staff`（使用者名稱）對應，**已有 `AdAccount` 欄位**。內容：約 25 個功能開關（`MapPN`、`DownloadBOM`、`AuthorityManagement` 等）、產品線（`ProductLine`、`ProductLinePermissions`）、模型層級（`CompleteSchematicProductLine`、`CompleteSchematicModel`、`OnlyViewModel`）、連線控管（`ConnectionLimited`、`OnlineCount`）、`AllowUsePNP1`、`ChangeSiteCode` |
| 權限檢查 | `BaseController.OnAuthorization` 每個請求都：重讀 `Authority`、檢查 `IsTokenActive`、`Alive()`（連線數上限），再以 `PortalAuthorization.PermissionToStringArray` 把功能開關轉成 Role，放進 `GenericPrincipal` |
| 使用範圍 | 程式中約 **317 處** `[Authorize(Roles=...)]` 或 `IsInRole`；資料層級檢查如 `PortalAuthorization.CheckProjectPermissions(authority, bookID, productCategory)`，需依 PNP 的資料判斷（例如某本書屬於哪條產品線） |
| 權限管理 | PNP 內的權限管理功能（`AuthorityManagement` 權限） |

---

## 3. 架構概念

```
                 身分驗證（維持現狀）                          授權（新）
使用者 ──▶ PNP 原本的登入頁（AD／LDAP，或 ucs\＋OTP）
                │
                │ Bridge／Redeem（沿用目前的機制）
                ▼
        Authorization Server ──────────────────────────▶ 授權中心（新模組）
                │                                         ・各系統的權限定義
                │ token／userinfo／權限查詢 API              ・AD 帳號（或 AD 群組）的權限指派
                ▼                                         ・權限管理、稽核
        PNP 網站（AD 使用者）、MCP Server、其他內部系統
```

- 身分驗證維持 PNP，**目前的 Bridge／Redeem／token／refresh 流程都可以沿用**。
- 授權是可以**獨立加上去的新模組**，不推翻現有成果。
- 外部人員繼續走 PNP 原本的 `Authority`，PNP 會長期並存兩條路徑。

---

## 4. 改動幅度（相對於目前已完成的成果）

### 4.1 Authorization Server：大（新增授權模組）

| 項目 | 內容 |
|---|---|
| 通用的權限資料模型 | 為了支援其他系統，**不可照搬** PNP `Authority` 表的欄位。建議：`應用程式（PNP、其他系統）` → `權限定義`（如 `pnp:MapPN`、`pnp:DownloadBOM`）→ `角色`（一組權限）→ `指派對象`（AD 帳號，日後可為 AD 群組）。PNP 的產品線、模型清單屬於「權限＋範圍值」（例如 `pnp:schematic` 限定產品線 A、B），需另行設計 |
| 身分識別碼 | 目前 token 的 `sub` 是 PNP 使用者名稱。跨系統使用時，`sub` 應改為**穩定的 AD 識別**（如 `網域\帳號` 或 AD objectGUID），並與外部人員的 `ucs\...` 區分 |
| 權限發放方式 | 粗粒度權限放在 token 的 claim；細粒度（產品線、模型清單）不適合放進 JWT（token 過大），改由 **OIDC 標準 userinfo 端點**提供（第5節採方式 A） |
| **OIDC 支援**（方式 A 需要） | 目前只簽發 access_token。PNP 網站要以 OIDC 取得「自己使用者」的身分與權限，需開放 `openid` scope、簽發 **id_token**，並啟用 **userinfo 端點**（OpenIddict 支援） |
| **Confidential client**（方式 A 需要） | 目前所有 client 都是 public（MCP Client 是使用者電腦上的程式）。PNP 網站是**伺服器端程式**，應註冊為 **confidential client**（持有 client secret，換 token 時須驗證），仍搭配 PKCE。註冊工具與 `RegistrationValidator` 目前固定為 public，需擴充（見第4.1.1節） |

#### 4.1.1 PNP 網站的 client 註冊（`pnp-web`）

PNP 網站與 MCP Client 一樣，要在註冊檔的 `clients` 新增一筆、以註冊工具寫入，但設定不同：

| | MCP Client（`pnp-mcp-client`） | PNP 網站（如 `pnp-web`） |
|---|---|---|
| 程式在哪裡執行 | 使用者的電腦（Claude Desktop 等） | **伺服器** |
| client 類型 | **public**：藏不住密碼，只靠 PKCE | **confidential**：持有 client secret，換 token 時驗證，並加上 PKCE |
| `redirect_uri` | `http://127.0.0.1:<port>/callback` 等本機網址 | PNP 自己的回呼網址，如 `https://pnp-server/PNP/oauth/callback` |
| 要取得什麼 | 呼叫 MCP Server 用的 access_token | **id_token、userinfo**（使用者身分與權限，給 PNP 自己使用） |
| scope | `ucs.*`、`pnp.*`、`offline_access` | `openid`、PNP 權限相關 scope、`offline_access` |
| 身分來源 | `pnp`（Bridge） | `pnp`（Bridge） |

註冊工具的擴充：

- 註冊檔可指定 client 類型（public／confidential）。
- **client secret 不可明文寫在註冊檔**（註冊檔放在版本控制）。比照 `hmacSecretRef`，註冊檔只寫設定 key，例如 `"clientSecretRef": "Clients:PnpWeb:Secret"`；註冊工具執行時從設定（user-secrets、環境變數、金鑰管理機制）讀出實際值寫入 DB。OpenIddict 會以雜湊保存，不存原文。
- 驗證規則：confidential client 必須有 secret 參照；`redirect_uri` 必須是 HTTPS（伺服器端不可用 loopback）。
- 同一組 secret 兩邊都要有：Authorization Server 由註冊工具寫入（雜湊），PNP 存在自己的設定（金鑰管理機制），換 token 時送出。
| refresh 時重新評估 | 權限調整後，於下次 refresh 生效 |
| 資料遷移 | 將 PNP `Authority` 中 **AD 使用者**的權限匯入（可利用既有的 `Authority.AdAccount` 對應） |
| 權限管理工具 | 人員權限異動頻繁，不同於 client 註冊（目前以 JSON 註冊檔管理），**很可能需要管理畫面**；此畫面為高價值攻擊目標，須有嚴格的存取控制與稽核 |
| 稽核 | 權限異動、權限查詢的紀錄 |
| 測試 | 權限模型、發放、遷移的自動化測試 |

### 4.2 PNP：中～大（採方式 A）

| 項目 | 內容 |
|---|---|
| redeem 回應 | 增加 **AD 帳號**與**使用者類型**（AD／外部）。PNP 登入時已以 LDAP 驗證 AD，可取得 |
| **OAuth／OIDC client** | PNP 網站成為 Authorization Server 的 confidential client：登入成功後導向 `/connect/authorize`、接收回呼、以 code＋`code_verifier`＋client secret 換 token、到期前以 refresh_token 換發（詳見第5.1節） |
| token 的保存 | token 存在**伺服器端**（Session 或伺服器端儲存），不放在瀏覽器 cookie；多主機部署時 Session 須共用 |
| `BaseController.OnAuthorization` | 分兩條路徑：AD 使用者使用 OAuth 取得的權限（存在伺服器端，不需每個請求查 DB）；外部人員照舊讀 `Authority` 表 |
| **以轉接器降低改動** | 從 OAuth 取得權限後，**組成與原本相同的 `Authority` 物件**，再沿用 `PermissionToStringArray` 轉成 Role。如此約 317 處 `IsInRole`／`[Authorize(Roles=...)]` 與 `CheckProjectPermissions` **都不需修改** |
| 維持不變 | `IsTokenActive`、`Alive()`（連線數上限）屬於登入狀態管理而非授權，留在 PNP |
| 權限管理畫面 | AD 使用者的權限改由授權中心管理；PNP 畫面只保留外部人員 |
| 切換開關 | 建議加入設定開關，可隨時切回舊機制 |

### 4.3 MCP Server：小

使用同一份權限（token claim 或 userinfo），主要影響 `pnp.*`（Native Capability API）。

### 4.4 文件

Authorization Server 的定位改變，定案後需調整設計文件第1、5、6、7、9節。

---

## 5. PNP 網站取得權限的方式

| | A. PNP 走標準 OAuth 流程 | B. PNP 呼叫權限查詢 API |
|---|---|---|
| 做法 | PNP 登入後，自己作為 OAuth client 跑一次授權流程。Bridge 認得 PNP 的登入 cookie，使用者**不會看到任何畫面**；取得 token 後由 token 或 userinfo 讀取權限 | PNP 以 HTTPS＋HMAC 呼叫 Authorization Server 的「查詢某 AD 帳號在 PNP 的權限」API（與 token-status 相同模式） |
| 標準程度 | ✅ 標準 OAuth／OIDC，其他系統也照此接入 | ⚠️ 自訂 API |
| 權限生效速度 | ⚠️ 等 token 換發 | ✅ 可每次查詢或短時間快取，接近現在的「立即生效」 |
| PNP 改動 | 較大（.NET Framework 4.7.2 需接 OIDC client） | 較小 |

**決定（2026-11-20）：採方式 A**。PNP 網站與日後接入的其他內部系統，一律以標準 OAuth／OIDC 流程接入授權中心，不另建自訂的權限查詢 API。

### 5.1 方式 A 的流程

```
1. 使用者在 PNP 原本的登入頁登入（AD／LDAP），PNP 簽發 .ASPXAUTH（維持現狀）
2. PNP 判斷為 AD 使用者 → 將瀏覽器導向 Authorization Server
     /connect/authorize?client_id=pnp-web&redirect_uri=<PNP 的回呼網址>&scope=openid ...&code_challenge=...
3. Authorization Server 第一次經過 → 導向 PNP /sso/bridge
4. PNP /sso/bridge：.ASPXAUTH 有效（剛登入）→ 發 bridgeCode → 導回 bridge-callback
5. Authorization Server：redeem → 第二次經過 /connect/authorize → 簽發 Authorization Code → 導回 PNP 回呼網址
6. PNP 回呼（伺服器端）：以 code＋code_verifier＋client secret 呼叫 /connect/token → 取得 id_token、access_token、refresh_token
7. PNP 由 id_token／userinfo 取得權限，存在伺服器端 → 轉接器組成 Authority 物件 → 進入原本的頁面
```

- 第2～6步全是瀏覽器自動導轉，使用者**不會看到任何額外畫面**（因為 PNP 剛登入，Bridge 直接通過）。
- **沿用目前已完成的 Bridge／Redeem 流程**：Authorization Server 不需要知道 PNP 登入頁的細節。
- 外部人員（`ucs\`＋OTP）登入後**不走第2步**，照舊使用 `Authority` 表。

### 5.2 方式 A 的注意事項

| 項目 | 說明 |
|---|---|
| 權限生效速度 | 權限在 token 換發時才更新：最慢約為 access_token 的效期。若需要更快，可為 PNP 這個 client 設定較短的 access_token 效期，或定期呼叫 userinfo 重新取得 |
| PNP 登出 | PNP 登出時 `UserToken.LogoutTime` 被設定，Authorization Server 下次 refresh 時的 `IsTokenActive` 檢查會拒絕並撤銷整條鏈（目前已實作）。PNP 端同時清除伺服器端保存的 token |
| 避免導轉迴圈 | `/sso/bridge` 與 OAuth 回呼端點本身**不可要求「已取得 OAuth 權限」**，否則會無限導轉；`BaseController` 對這兩個端點須排除 |
| .NET Framework 4.7.2 的實作方式 | 可用 OWIN 的 OpenID Connect middleware（`Microsoft.Owin.Security.OpenIdConnect`），但它預設搭配 OWIN cookie 驗證，與 PNP 現有的 `FormsAuthentication` 整合不易；另一做法是**自行實作授權碼流程**（一個導向動作＋一個回呼 action＋以 `HttpClient` 換 token），範圍明確、不影響現有登入。實作前再評估 |
| client secret 的保存 | PNP 的 client secret 與 HMAC 密鑰同等級，須以金鑰管理機制保存，不可明文寫在 `Web.config` |
| Authorization Server 停機 | AD 使用者無法完成第2～6步而無法使用 PNP。需決定是否允許暫時退回舊機制（`Authority` 表），或要求 Authorization Server 高可用 |

---

## 6. 由 OAuth 集中管理授權 vs PNP 自行控管

| | 由 OAuth 集中管理 | PNP 自行控管（現況） |
|---|---|---|
| 跨系統 | ✅ 一個人在所有系統的權限，在同一處管理、查詢、稽核 | ❌ 各系統各一套，離職、調職需逐一處理 |
| 使用者識別 | ✅ 以 AD 帳號為準，跨系統一致 | ⚠️ 各系統自己的帳號名稱 |
| 權限生效速度 | ⚠️ 方式 A：token 換發時才更新（可用較短的 token 效期或定期呼叫 userinfo 縮短） | ✅ 每個請求重讀，立即生效 |
| 強制登出／踢人 | 仍由 PNP 的 `IsTokenActive` 處理（不在授權中心範圍） | ✅ |
| 連線數上限 | OAuth 無此概念，仍由 PNP 的 `Alive()` 處理 | ✅ |
| 資料層級檢查 | ⚠️ OAuth 只提供「可看哪些產品線」，比對「這本書屬於哪條產品線」仍在 PNP（以轉接器沿用既有程式） | ✅ |
| 效能 | ✅ 權限可放在 token 或快取 | ⚠️ 每個請求查 DB |
| 可用性 | ⚠️ Authorization Server 成為**所有接入系統的關鍵依賴**：停機時 AD 使用者無法取得權限，需高可用部署或在各系統做短期快取 | ✅ PNP 自主 |
| 安全集中度 | ⚠️ Authorization Server 與權限管理畫面被攻破即影響所有系統，需更嚴格的保護與稽核 | 影響範圍只有 PNP |
| 管理責任 | ⚠️ 需明確由誰維護各系統的權限（集中單位，或各系統管理員在授權中心管理自己的系統） | 各系統自行維護 |
| 機制數量 | ⚠️ PNP 長期並存兩條路徑（AD 使用者走 OAuth、外部人員走 `Authority`） | 一套 |

**結論**：以「多系統、單一授權管理」的長期目標而言，由 OAuth 管理授權是正確方向。代價是 Authorization Server 的重要性與安全要求大幅提高，並需新增「權限管理」這個全新模組；PNP 端以轉接器可把改動集中在 `BaseController` 附近。

適合交由 OAuth 發放的是**粗粒度、變動不頻繁**的權限；**即時生效、細粒度、與資料綁定**的部分，仍需在擁有資料的系統內完成判斷。

---

## 7. 建議的進行順序

| 階段 | 內容 |
|---|---|
| 0. 設計 | 通用權限模型、AD 識別碼格式、redeem 新格式、權限放在 id_token／access_token／userinfo 的分配、權限由誰管理、PNP 端的 OIDC 實作方式（第5.2節） |
| 1. Authorization Server | 權限資料模型、自 `Authority` 匯入 AD 使用者；OIDC 支援（`openid` scope、id_token、userinfo 端點）；confidential client 支援（註冊工具、驗證規則一併擴充）；**此時尚無系統使用** |
| 2. PNP | redeem 回傳 AD 帳號與使用者類型；PNP 成為 OIDC client（導向、回呼、換 token、refresh、伺服器端保存）；`BaseController` 對 AD 使用者改用 OAuth 取得的權限（轉接器），附切換開關 |
| 3. 管理 | AD 使用者的權限管理移至授權中心；PNP 管理畫面只保留外部人員 |
| 4. 擴大 | MCP 的 `pnp.*` 使用同一份權限；其他內部系統以相同的標準 OAuth／OIDC 流程接入，身分驗證**不經過 PNP 的 Bridge**，改由 Authorization Server 直接向公司 AD 確認（第10節） |

---

## 8. 與目前進行中工作的關係

目前進行中的 MCP Resource Server 試做（`ucs.*`，以 `bomNo` 查詢 ECN/RCN）使用系統層級的 UCS 帳號、**沒有 per-user 權限判斷**，與本規劃互不影響，可先行進行。本規劃的「階段 0：設計」可同時進行（主要為討論與決策）。

兩個工作在以下時間點交會，屆時需先確認本規劃的設計是否定案：

| 交會點 | 原因 | 時機 |
|---|---|---|
| redeem 的回應格式 | 授權中心需要 redeem 增加 AD 帳號與使用者類型；若 PNP 正式 SSO 端點先以目前格式實作，之後需再改一次 | PNP 實作正式 SSO 端點（取代 MockPnp）**之前** |
| token 的 `sub` 格式 | 授權中心建議改為穩定的 AD 識別；`ucs.*` 試做只用 `sub` 寫 log，影響小；`pnp.*` 會用 `sub` 做權限判斷 | 實作 `pnp.*`（Native Capability API）**之前** |

```
現在 ─┬─ MCP 試做（PNP 內部 API、MCP Server、ECN/RCN Tool）
      └─ 本規劃「階段 0：設計」（討論與決策）
              │ 定案：權限模型、AD 識別碼格式、redeem 新格式、權限發放方式
              ▼
之後 ─── PNP 正式 SSO 端點 ← 以新的 redeem 格式實作
      ─── 本規劃階段 1～3
      ─── MCP 的 pnp.* 能力 ← 使用授權中心的權限
```

若本規劃有明確期限或業務優先順序較高，可調整為優先進行。

---

## 9. 待確認事項

| 項目 | 影響 |
|---|---|
| 權限指派給**個人 AD 帳號**，或日後使用 **AD 群組** | 群組可大幅簡化多系統管理，但 Authorization Server 需能查詢 AD（LDAP 存取權限） |
| 權限調整後**多快**需生效 | 決定發放方式（token／查詢 API）與快取時間 |
| 集中後**由誰**維護權限 | 集中管理單位，或各系統管理員各自管理；影響管理畫面的權限設計 |
| Authorization Server 的**可用性**要求 | 成為多系統關鍵依賴後，需評估高可用部署 |
| AD 識別碼採用哪個屬性 | `網域\帳號`、UPN 或 objectGUID（帳號更名時是否需維持同一身分） |
| PNP 其他系統管理單位的配合 | 登入頁由其他部門維護：確認登入後取得 AD 帳號、以及登入成功後導向 Authorization Server（第5.1節第2步）的方式，不需修改登入頁本身 |
| PNP 端 OIDC 的實作方式 | OWIN OpenID Connect middleware，或自行實作授權碼流程（第5.2節） |
| PNP 的 Session 與多主機 | token 存在伺服器端；PNP 若為多主機部署，Session 須共用 |
| Authorization Server 停機時的處理 | 是否允許 AD 使用者暫時退回 `Authority` 表 |
| 新系統的 AD 驗證方式 | 聯邦（公司 SSO）、Windows 整合驗證（Negotiate）、直接驗證（LDAP）三選一或組合（第10.4節）；需先向 IT 確認第10.5節的問題 |

---

## 10. 新系統的身分驗證：不經過 PNP 的 Bridge

### 10.1 為什麼新系統不走 Bridge

Bridge 是為 PNP 設計的：身分驗證寄放在 PNP 的登入頁（登入頁由其他部門維護，必須保留）。若其他部門的系統也走 Bridge，就會變成「登入任何系統都依賴 PNP 的登入頁」：PNP 停機，所有系統都無法登入，也不合理。

長期而言，新系統應由 **Authorization Server 直接向公司 AD 確認身分**。注意：不是「有 AD 帳號就能換到 token」，使用者仍須**證明自己是該 AD 帳號的主人**，只是證明的方式不再經過 PNP。

### 10.2 依 client 選擇身分來源（沿用既有機制）

設計文件第6節定義了身分來源的模式，目前只實作 Bridge。既有的「依 `client_id` 決定身分來源」機制（`OpenIddictApplications.Properties.identity_provider_id` → `IdentityProviders`）可直接沿用：新系統的 client 綁定新的身分來源（例如 `company-ad`），`/connect/authorize` 就改走 AD 驗證，完全不經過 PNP。

```
pnp-web、pnp-mcp-client  → identity_provider_id = "pnp"         → Bridge（PNP 登入頁）
新系統 A、新系統 B        → identity_provider_id = "company-ad"  → AD 驗證（第10.4節）
```

### 10.3 關鍵：身分識別碼必須一致

同一個人透過 PNP（Bridge）登入，與透過新系統（AD 驗證）登入，token 的 `sub` **必須是同一個值**，授權中心才能把他視為同一人、套用同一份權限。因此第4.1節建議 `sub` 改為穩定的 AD 識別：

- Bridge 路徑：redeem 回傳 AD 帳號（與識別碼），由 Authorization Server 轉成統一格式。
- AD 驗證路徑：本來就取得 AD 身分（帳號、SID 等）。
- 兩邊對齊成同一種格式（建議以 SID 或 objectGUID 這類不因改名而變的識別碼為主，帳號名稱為輔）。

### 10.4 三種 AD 驗證方式

#### 聯邦模式（公司 SSO）

Authorization Server 本身成為公司 SSO（ADFS、Entra ID 等）的一個 client，由公司 SSO 確認身分後回傳使用者資料：

```
使用者 → 新系統 → Authorization Server
                        │ 請公司 SSO 確認身分
                        ▼
                  公司 SSO（ADFS／Entra ID，由 IT 管理）
                        │ 使用者在此登入（或已登入，直接通過）
                        ▼
                  回傳身分資料（AD 帳號、識別碼、群組）
                        │
                        ▼
Authorization Server：對應授權中心的權限 → 簽發 token 給新系統
```

- **需要 IT 配合**：在公司 SSO 註冊 Authorization Server（ADFS 的信賴憑證者、Entra ID 的應用程式註冊），取得 `client_id` 與 secret（或憑證）；登記 Authorization Server 的回呼網址；設定回傳的使用者屬性（`sAMAccountName`／UPN、objectGUID、AD 群組）；網路連線；IT 的申請與資安審核。
- **我們要做的**：實作設計文件第6節的「聯邦模式」，以 OIDC client 身分串接公司 SSO（ASP.NET Core、OpenIddict 皆有現成套件），再將 AD 身分對應到授權中心的權限。不需登入頁，**不經手使用者密碼**。
- **額外好處**：公司 SSO 的安全政策自動套用，例如多因素驗證（MFA）、條件式存取（限公司網路或公司電腦）、帳號停用即無法登入。

#### Windows 整合驗證（Negotiate：Kerberos，必要時退回 NTLM）

使用者以加入網域的公司電腦登入 Windows 後，瀏覽器自動以目前的 Windows 身分完成驗證，**完全不需輸入帳號密碼**。

```csharp
// Authorization Server 的 Program.cs
builder.Services.AddAuthentication(NegotiateDefaults.AuthenticationScheme)
    .AddNegotiate();
```

驗證通過後可取得 `DOMAIN\帳號`、使用者的 **SID**（適合作為 `sub`）與**所屬 AD 群組**（適合用於權限指派）。

```
新系統的 client（identity_provider_id = "company-ad"）
    → /connect/authorize
    → Authorization Server 以 Negotiate 驗證瀏覽器（不顯示登入頁、不經手密碼）
    → 取得 DOMAIN\帳號、SID、AD 群組
    → 對應授權中心的權限 → 簽發 token
```

「不需輸入密碼」須同時滿足：

| 條件 | 說明 |
|---|---|
| 使用者電腦已加入公司網域，並以 AD 帳號登入 Windows | 個人電腦、手機、未加入網域的電腦不行 |
| 瀏覽器信任該網站 | 網站須在「近端內部網路」區域或允許清單，通常由 IT 以群組原則統一設定；否則會跳出帳密視窗 |
| 伺服器端設定 | IIS 啟用 Windows 驗證，或 Kestrel 使用 Negotiate |
| Kerberos 需要 SPN | 由 IT 為網站的主機名稱與執行帳號註冊 SPN；沒有時退回 NTLM |

限制：

| 項目 | 說明 |
|---|---|
| NTLM 的安全性 | NTLM 是舊協定，有轉送攻擊（relay）風險，微軟已逐步淘汰。應盡量走 **Kerberos**（請 IT 註冊 SPN），並一定搭配 HTTPS |
| 公司外部或非公司電腦 | 無法使用，需替代方案（帳密視窗或另提供登入頁） |
| 外部人員 | `ucs\` 帳號不是 AD 帳號，繼續走 PNP 的 Bridge |
| 只在瀏覽器階段 | OAuth 的登入本來就在瀏覽器完成，沒有問題；MCP Client（Claude Code 等）同樣開瀏覽器登入。取得 token 後的 API 呼叫使用 token，不再需要 Windows 驗證 |
| 仍需 IT 配合 | SPN 註冊、瀏覽器群組原則（比聯邦模式少） |

#### Windows 整合驗證在 OAuth 流程中的位置

Windows 整合驗證不是在 OAuth 流程**之前**先做，而是**在 OAuth 流程裡面**，取代 Bridge 那一段。`/connect/authorize` 第一次被呼叫時，Authorization Server 必須先知道使用者是誰才能簽發 Authorization Code；這一步「怎麼確認身分」，由 client 綁定的身分來源決定：

```
                  Bridge（PNP 的 client）                     Windows 整合驗證（新系統的 client）
                  ──────────────────────                     ──────────────────────────────
1. client 導向     /connect/authorize                         /connect/authorize
2. 確認身分        導去 PNP /sso/bridge                        Authorization Server 直接要求瀏覽器提供 Windows 身分
                  → PNP 檢查登入 cookie                         → 瀏覽器自動送出 Kerberos 票證（使用者看不到）
                  → bridgeCode 導回 bridge-callback             → Authorization Server 得知 DOMAIN\帳號、SID、群組
                  → redeem 換回身分
                  → 繞回 /connect/authorize（第二次經過）
3. 簽發 code       導回 client 的 redirect_uri?code=...          導回 client 的 redirect_uri?code=...
4. 換 token        client 呼叫 /connect/token                   client 呼叫 /connect/token
```

- **第1、3、4步完全相同**，都是標準 OAuth 流程；只有**第2步「怎麼確認身分」不同**。
  - Bridge：繞去 PNP、換碼、再繞回來，需要多次導轉與「第二次經過」。
  - Windows 整合驗證：在 `/connect/authorize` **當下**完成，不需繞去其他地方，也不需要「第二次經過」。
- **對 client 而言沒有差別**：新系統（或 MCP Client）只知道「導去 `/connect/authorize`，最後拿到 code、換到 token」，不需要知道 Authorization Server 中間用哪種方式確認身分。
- 這正是設計文件第6節「身分來源路由」的目的：**核心 OAuth 流程不變，只抽換「確認身分」的方式**。聯邦模式、直接驗證模式同理，都是替換第2步。

#### 直接驗證模式（LDAP 帳密）

Authorization Server 自己提供登入頁，以 LDAP 向 AD 驗證使用者輸入的帳密。

- **需要 IT 配合**：查詢 AD 用的服務帳號、Authorization Server 連得到網域控制站（LDAP／LDAPS port）、LDAPS 憑證。
- **我們要做的**：登入頁、密碼處理（防暴力破解、記錄、資安規範）、MFA 等安全措施都要自行實作。

#### 比較

| | 直接驗證（LDAP 帳密） | Windows 整合驗證（Negotiate） | 聯邦（公司 SSO） |
|---|---|---|---|
| 使用者要輸入密碼嗎 | 要 | **不用**（公司電腦、公司網路） | 視公司 SSO 設定，已登入通常不用 |
| Authorization Server 經手密碼 | ✅ 要（風險最高） | ❌ 不用 | ❌ 不用 |
| 登入頁 | 要自己做 | 不用 | 不用 |
| MFA、條件式存取 | 要自己做 | 沒有（只依賴 Windows 登入） | 由公司 SSO 提供 |
| 公司外部或非公司電腦 | 可以 | 不行，需替代方案 | 視公司 SSO 而定 |
| 可取得 AD 群組 | 需另外查詢 AD | ✅ 直接取得 | 視 IT 設定的屬性 |
| 需要 IT 配合的程度 | 低（LDAP 服務帳號） | 中（SPN、瀏覽器原則） | 高（應用程式註冊、屬性設定） |

**建議**：公司若已有 SSO，**聯邦模式**為首選（最安全、不經手密碼、自動套用公司安全政策）。若沒有或短期內申請不到，**Windows 整合驗證**是很實際的選擇（使用者體驗最好、不經手密碼）。直接驗證（LDAP 帳密）僅作為最後選項，或作為非公司電腦的替代方案。

### 10.5 建議先問 IT 的問題

1. 公司有沒有既有的 SSO？使用 ADFS、Entra ID，還是其他系統？
2. 內部系統接入 SSO 的申請流程？
3. 能提供哪些使用者屬性：帳號、UPN、objectGUID、SID、**AD 群組**？
4. 是否要求 MFA 或條件式存取？
5. 若採 Windows 整合驗證：能否為 Authorization Server 註冊 SPN？瀏覽器的「近端內部網路」群組原則由誰設定？
6. 若採 LDAP：能否提供查詢 AD 的服務帳號與 LDAPS 連線？
