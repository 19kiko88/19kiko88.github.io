---
title: 建立OAuth Server with OpenIddict(2)
date: 2026-10-27 17:44:03
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
## 資安加固 ：限定aud（audience）claim

這一步在做什麼：確保JWT的aud（受眾/目標對象）明確寫上「這個token只給某個特定的Resource Server用」，而不是一個沒有限定範圍的通用token。OpenIddict的做法是：把「資源（resource）」關聯在scope上，當某個scope被授予時，OpenIddict會自動把該scope關聯的resource加進aud裡，不需要我們手動在Controller程式碼裡額外處理。

- **修改Step 7的seed程式碼，幫每個scope加上Resources**

在Program.cs的SeedDataAsync()裡，把scope的定義從：
``` csharp
var scopeDescriptors = new[]
{
    new OpenIddictScopeDescriptor { Name = "ucs.sn-info.read", DisplayName = "SN 資料讀取" },
    new OpenIddictScopeDescriptor { Name = "ucs.ecn-rcn.read", DisplayName = "UCS ECN/RCN 資料讀取" },
    new OpenIddictScopeDescriptor { Name = "pnp.auto-open-pnp.read", DisplayName = "SN自動開啟點圖 讀取" },
};
```
改成每個都加Resources：
``` csharp
const string mcpResourceServerId = "urn:pnp:ucs-mcp"; // TODO：等真正的MCP Resource Server確定網址/識別碼後再更新

var scopeDescriptors = new[]
{
    new OpenIddictScopeDescriptor { Name = "ucs.sn-info.read", DisplayName = "SN 資料讀取", Resources = { mcpResourceServerId } },
    new OpenIddictScopeDescriptor { Name = "ucs.ecn-rcn.read", DisplayName = "UCS ECN/RCN 資料讀取", Resources = { mcpResourceServerId } },
    new OpenIddictScopeDescriptor { Name = "pnp.auto-open-pnp.read", DisplayName = "SN自動開啟點圖 讀取", Resources = { mcpResourceServerId } },
};
```

說明：mcpResourceServerId目前先用一個佔位的URN字串（urn:pnp:ucs-mcp），代表「包裝UcsService的那個MCP Resource Server」。之後MCP Resource Server真的做出來、有實際網址後，要回來更新這個值，並且兩邊要一致——MCP Resource Server驗證JWT時，會檢查aud是不是等於這個值，兩邊對不上就會拒絕。

為什麼這樣設定後aud就會自動出現：OpenIddict簽發token時，會看這次授權的scope裡關聯了哪些resource，自動把這些resource值組進JWT的aud欄位，不需要我們在CompleteAuthorizationAsync()裡額外呼叫任何方法。

- **重建scope資料讓新設定生效**
因為scope的seed邏輯是「查無這個名稱的scope才建立」，既有的3筆scope資料不會因為程式碼改了就自動更新，一樣要先手動刪除：
``` sql
DELETE FROM "OpenIddictScopes" WHERE "Name" IN ('ucs.sn-info.read', 'ucs.ecn-rcn.read', 'pnp.auto-open-pnp.read');
```

刪完重新F5跑起來，<span style="color: red;">讓seed重新建立這3筆（這次會帶著Resources）。</span>

請執行後確認：
1. dotnet build正常編譯。
2. 重新完整走一次流程拿新的access_token，丟去jwt.io解開，確認payload裡出現aud欄位，值是urn:pnp:ucs-mcp。