---
title: Claude Skill
categories:
  - AI應用
tags:
  - AI
  - Claude Skill
  - Claude Code
  - 資安
--- 
##### 最近剛改完弱點掃描。為了避免日後每年都要痛苦一次，除了先前提到的使用CLAUDE.md + SECURITY.md來讓Claude Code在產出程式時，遵守相關的資安規範，另外也透過了Claude Skill來建立相關的弱掃知識庫。
<!-- more -->
#
#
#
#
---
### 什麼是 Claude Skill
傳統的 AI Agent 使用方式是：每次下 prompt 給 Agent，請它依照描述內容執行任務。如果某個任務需要週期性重複執行，我們就得每次重新輸入相同的 prompt。

有了 Claude Skill 後，我們可以把常重複使用的工作流程，包裝成一個獨立的 Skill 資料夾，裡面包含任務說明（SKILL.md）、使用方式、相關參考資料、甚至可執行的腳本等。使用時只需要下簡單的 prompt，Claude 就會自動比對、載入對應的 Skill，再透過裡面的說明與參考資料完成任務。而且 Claude 只會在需要時才載入 Skill 的詳細內容，不會一次把所有 Skill 都塞進上下文，因此即使建立很多 Skill，也不會拖累回應效率。

Claude Skill 採用<span style="color:red;">分層載入</span>的方式讀取參考資料：SKILL.md 是第一層（摘要、導覽），reference 檔案是第二層（詳細知識），Claude 依任務需求逐層深入讀取，而不是一次把所有內容都載入。可以把它簡單理解為參考資料的<span style="color:red;">「索引化」</span>——透過一層一層的目錄索引，找到對應的知識內容，再回應給使用者。

**把工作流程「Skill 化」主要有這幾個優點：**
1. 不用重複下 prompt
把常用的工作流程包成 Skill 後，之後只要簡單說一句需求，Claude 自動觸發，不用每次複製貼上一大段指令。
2. 輸出品質穩定
因為流程、規範、參考資料都已經定義好放在 Skill 裡，每次執行的結果會一致，不會因為你當下 prompt 寫得好不好而影響品質。
3. 節省上下文
Claude 只在需要時才載入 Skill 的詳細內容，平常只記住「這個 Skill 存在」這件事，所以就算你建了很多 Skill，也不會互相干擾或拖慢回應。
4. 知識可以索引化
透過分層載入的設計，可以把大量的參考資料、SOP、規範都收進 Skill 裡，Claude 需要時才讀取對應的那份，不會一次全塞進去。
5. 可重複使用、團隊共享
Skill 建立一次，所有對話都能用，甚至可以讓整個團隊共用同一套 Skill，確保大家執行流程一致。
#
#
#
---
### 如何建立一個Skill
1. ##### 建立Skill資料夾
``` text
my-skill/
└── SKILL.md
```
##### SKILL.md 的基本格式：
* name 與資料夾名稱一致，小寫＋dash 格式正確。
* description（389 bytes，遠低於 1024 字元上限）
#
``` text
---
name: my-skill
description: 說明這個 skill 做什麼，以及什麼時候觸發
---

# My Skill

## 執行步驟
1. 第一步...
2. 第二步...
```
#
#
2. ##### 需要擴充時再加資料夾：
``` text
my-skill/
├── SKILL.md          ← 必要
├── references/       ← 需要知識庫時加
├── scripts/          ← 需要執行腳本時加
├── assets/           ← 需要範本時加
└── evals/            ← 需要測試時加
```

* ##### 重點：
  * ##### description 寫清楚，決定 Claude 何時觸發
  * ##### SKILL.md 盡量保持在 500 行內，細節移到 references/
  * ##### 有幾個 skill 就建幾個子資料夾，每個各有自己的 SKILL.md
#
#
#
---
### 實際範例
##### 看完上面的範例後，再讓我們來看看實際用在專案上的Skill資料夾結構，這個Skill是關於弱點修補知識庫 skill。VULN_REF/ 下每個資料夾對應一種具體弱點，裡面的 *_Fix_Report.md 就是該弱點的標準修補說明。當 Claude 掃描到特定弱點時，會去查對應的修補文件，給出針對性的建議。
{% asset_image skill_structure.jpg skill_structure %}
#
#
##### 實際的使用流程：
##### 做了一段有SQL_Injection弱點的範例程式
``` csharp
        [AllowAnonymous]
        [HttpGet]
        public ActionResult SqlInjectionDemo(string keyword)
        {
            var rows = new List<string>();
            string errCode = "0";
            try
            {
                string connStr = System.Configuration.ConfigurationManager.ConnectionStrings["MyDB"].ConnectionString;

                string sql = "SELECT \"EmployeeID\", \"Name\" FROM \"UserInfo\" "
                           + "WHERE \"Name\" LIKE '%" + keyword + "%'";

                using (var cn = new System.Data.SqlClient.SqlConnection(connStr))
                {
                    cn.Open();
                    using (var cmd = new System.Data.SqlClient.SqlCommand(sql, cn))
                    using (var reader = cmd.ExecuteReader())
                    {
                        while (reader.Read())
                            rows.Add(reader["EmployeeID"] + ":" + reader["Name"]);
                    }
                }
            }
            catch (Exception ex)
            {
                errCode = "1";
            }
            return Content(string.Join(",", rows) + ";" + errCode);
        }
```
  1. ##### 在Claude Code下prompt。
  ``` bash
  .\Controllers\AccountController.cs的SqlInjectionDemo被AppScan DAST發現SQL_Injection，幫我用skill修正。
  ```
  #
  #
  2. ##### Claude Code依照Skill的指令開始處理這個弱點 
  {% asset_image skill_run1.jpg skill_run1 %}
  #
  #
  ##### 可以看到，Claude讀取SKILL.md後，再依序找到對應的dast-guide.md，再從VULN_REF目錄裡找到對應的修復報告，再依照規範來修改SQL_Injection弱點。
  #
  #
補充：<span style="color:red;">要如何下prompt，才能觸發Skill</span>
* .\Controllers\AccountController.cs的SqlInjectionDemo被偵測到SQL_Injecttion弱點。幫我修正。 => 會被直接修正，會是透過SECURITY.md的規範來修正。
* .\Controllers\AccountController.cs的SqlInjectionDemo被AppScan DAST發現SQL_Injection，幫我用skill修正。 => 會觸發Skill規範進行修正。

CLAUDE.md 是專案記憶檔，每次對話一開始就會被自動載入，而且這條規則是無條件的——只要偵測到「要改程式碼」，不管跟 AppScan、弱點掃描有沒有關係，都會先讀 SECURITY.md。

Skill的觸發依據就是 description 裡寫的條件。
``` text
name: security-scan
description: 適用於 PCBRepairNavi（ASP.NET MVC 5）專案。當使用者提供 AppScan DAST 黑箱弱掃報告（PDF 或弱點名稱，如 SQL Injection、CSRF、CSP 缺漏等），或要求修復掃描出的弱點時使用本技能：解讀弱點偵測手法、定位受影響程式碼、套用修正並產出格式一致的修正報告。不適用於 SAST（靜態分析）報告。
```
#
#
所以貼近這些<span style="color:red;">強訊號</span>關鍵詞最有效：
  * 提到「Skill」「AppScan」「DAST」「弱掃報告」
  * 直接貼 PDF 報告路徑，例如 202606弱掃/黑箱/api/SQL_Injection.pdf
#
#  
相反的，像下面這種<span style="color:red;">弱訊號</span>就容易只落到 CLAUDE.md 而不觸發 skill：
``` text
幫我改一下這段程式碼的SQL Injection問題
```
這種說法太泛，Claude 只會判定「要改 code → 先讀 SECURITY.md」，不會特別聯想到這是 AppScan 報告修復工作。
#
#
最保險的做法：如果你想確定一定會用到這個 skill，可以直接點名：
``` text
請用 security-scan 這個 skill 幫我分析並修正這份弱掃報告
```
明確講出 skill 名稱，Claude 看到跟它索引到的 skill 名稱一致，基本上就不會誤判。
#
#
#
---
### 加碼拿掉Claude Skill處理SQL_Injection的結果：
  1. ##### 在Claude Code下prompt。
  ``` bash
  .\Controllers\TestController.cs的SqlInjectionDemo被偵測到SQL_Injecttion弱點
  ```
  #
  #
  2. ##### Claude Code依照SECURITY.md的規範開始處理這個弱點 
  {% asset_image no_skill_run1.jpg no_skill_run1 %}
  #
  #
  {% asset_image no_skill_run2.jpg no_skill_run2 %}
  #
  #
  ##### 可以看到，把Skills資料夾砍掉後，Claude改讀取SECURITY.md裡的規範來修改SQL_Injection弱點。<span style="color:red;">這邊我也順便測試到了透過CLAUDE.md引用的SECURITY.md真的有啟動~</span>
#
#
---
### CLAUDE.md跟Skill.md的使用時機

前面的範例中，CLAUDE.md 跟 SKILL.md 規範了類似的內容，可能會讓人疑惑：既然用起來效果差不多，那什麼時候該用 CLAUDE.md？什麼時候該用 SKILL.md？

簡單用一個問題來判斷：**「這件事是每次對話都需要（主動）執行，還是只有被要求時才（被動）執行？」**

1. 每次都需要 → **CLAUDE.md**（例如：程式碼風格、專案慣例）
2. 被要求才需要 → **SKILL.md**（例如：資安掃描、產生報告）

CLAUDE.md 的內容會在每次對話一開始就全部載入，屬於「常駐」的規範；SKILL.md 則是 Claude 判斷任務相關時才會載入，平常不會占用上下文，適合放不常用、但需要時要很完整的專業流程。

<span style="color:red;">以我這個專案為例，同樣是「主動 vs 被動」的原則：AI 寫 code 時會主動遵守規範，人工寫 code 則可能遺漏、只能事後被動抓漏。

1. **CLAUDE.md** → AI 產出的 code。AI 每次寫 code 時都會主動套用 CLAUDE.md 裡的資安規範，不需要額外要求。
2. **SKILL.md** → 人工產出的 code。人工開發時有時會忘記套用資安規範，事後才需要「被要求」去掃描、抓出弱點並修正，正好符合 SKILL.md「被動、被要求才執行」的定位。
</span>
#
#
#
---
### CLAUDE CODE對這份Skill的分析結果(僅供參考)：

prompt：
``` text
我已經做完修正了，在幫我重新分析.claude\security-scan這個Skill的目錄架構以及md檔的內容，是否符合Claude Skill的精神以及標準架構
```

{% asset_image skill_analyze1.jpg skill_analyze1 %}
{% asset_image skill_analyze2.jpg skill_analyze2 %}
#
#
---
### 補充
今天把之前改的弱掃提交副測，又多了一個弱點，透過Skill輕鬆修復。
{% asset_image example1.jpg example1 %}
---
### REF：
1. [Agent skill 是什麼？Agent skill教學，6步驟打造你的第一個Skill](https://www.bnext.com.tw/article/90058/agent-skills-free-course-deeplearning-ai-anthropic-latest-partnership) 
2. [應用Claude Code進行弱點掃描修改](https://19kiko88.github.io/2026/06/17/%E6%87%89%E7%94%A8claude-code%E9%80%B2%E8%A1%8C%E5%BC%B1%E9%BB%9E%E6%8E%83%E6%8F%8F%E4%BF%AE%E6%94%B9/)

