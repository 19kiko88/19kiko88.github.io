---
title: Angular Component 設計檢討
date: 2026-08-21 17:57:24
categories:
 - 前端開發
tags:
 - Angular
 - AI產出
---
Angular Component 設計檢討 —— 從 app.component.ts 學到的教訓
<span style="color:red;">文章內容透過ai整理產出</span>
<!-- more -->


本文件整理自對本專案 `src/app/app.component.ts` 的檢視與重構過程，記錄實際發現的設計問題、根因，以及對應的改善做法，目的是讓未來開發新 Angular 專案時，能在設計初期就避開這些問題，而不是累積成技術債後才處理。

所有範例都來自本專案真實的程式碼與實際重構結果，不是憑空舉例。

---

## 現況：`app.component.ts` 的問題清單

在重構前，`app.component.ts` 呈現以下症狀（截至本文件撰寫時，仍有部分未處理）：

- 建構子一次注入 **26 個 service**
- 單一方法（`ngOnInit`、`getApiData`、`init`）動輒 80-150 行，混雜商業邏輯、DOM 操作、非同步 API 排程
- 元件內直接呼叫 `document.querySelectorAll`、`element.addEventListener`、`element.setAttribute`
- 完全沒有具備真實驗證價值的單元測試——所有既有測試都只是 `expect(x).toBeTruthy()` 的樣板，且大多數連編譯執行都做不到
- 留有大段被註解掉的死程式碼（`MutationObserver` 相關區塊）
- Class field 在宣告時就直接讀取建構子注入的 service（`barcodeSn = this._mvcService.indexViewModel.BarcodeSN;`），執行順序隱性依賴 TypeScript 編譯規則

以下逐項說明問題本質、為什麼會發生，以及新專案該怎麼做。

---

## 1. 建構子注入過多 Service（God Component）

### 症狀

`AppComponent` 的建構子注入了 `KnowledgeBaseService`、`SvgService`、`MvcService`、`ConfigService`、`PartsRankingService`、`TutorialVideoService`、`ContextMenuService`、`NavBarService`、`ModalService`、`ApiService`、`ToolTipService`、`VirtualGroupService`、`RightControlPanelService`、`ChipsetRepairService`、`PowerFunctionService`、`UcsService`、`LanguageService`、`UtilityService`、`StatusBarService`、`LayerService`、`ThemeService`、`SearchService`……等 26 個依賴。

### 為什麼不好

- 這是單一職責原則（SRP）被違反的直接徵兆：一個類別擁有的理由越多，越難維護、越難測試、越容易在修改時牽動不相關的功能。
- 每加一個依賴，測試這個元件時就要多想一種 mock/stub 的策略。26 個依賴意味著光是寫一個「should create」的測試，就要應付 26 條依賴鏈上可能出現的建構子副作用（本專案就實際發生過：光是讓測試套件能編譯執行，就必須建立一整套 `BASE_TEST_PROVIDERS` 去餵這些依賴需要的 `MvcService`/`HttpClient`/`TranslateService`）。
- 這種累積通常不是一次性造成的，而是「每次加新功能，最方便的整合點就是往根元件多塞一個 service」，日積月累的結果。

### 新專案該怎麼做

- **App 根元件應該儘量「薄」**：只負責畫面骨架（`<router-outlet>`、全域 layout）與少量、明確的啟動觸發點，不該是全 app 商業邏輯的匯集點。
- **把「初始化流程」本身封裝成一個專門的 Bootstrap/Orchestrator Service**，例如 `AppInitializerService`，讓它去注入需要協調的各種 service，根元件只呼叫 `appInitializerService.init()` 一個方法。這樣元件的建構子依賴數量會大幅下降，而且這個 orchestrator service 更容易單獨測試（因為它不用處理 Angular 元件生命週期、`ViewChild`、DOM 等問題）。
- 如果某個 service 只在元件裡被使用一次、且用途單一，優先考慮這個依賴是否該屬於另一個「已經被注入」的 service，而不是讓元件直接持有它（詳見第 6 節「移除依賴前要檢查循環風險」）。

---

## 2. 商業邏輯寫在 Component，而不是 Service 或純函數

### Angular 的慣例

> **Component 負責畫面綁定（wiring）；Service 負責邏輯；純粹的決策邏輯（無副作用）應該是獨立的、可單獨測試的函數。**

Component 的職責應該止於：接收使用者事件、呼叫 service、把 service 回傳的資料綁定到畫面。任何「決定要不要做某件事」的判斷邏輯，一旦超過一行、有明確的輸入輸出，就該被抽出。

### 本專案的實際案例

在 `app.component.ts` 的 `init()` 方法裡，曾經有這樣的程式碼直接寫在元件裡：

```ts
// 修改前：判斷邏輯與副作用混在一起，寫在元件裡
if (
  this._mvcService.projectInfo.bomNumber.startsWith('60YV0ID7') ||
  this._mvcService.projectInfo.bomNumber.startsWith('60YV0JS0') ||
  this._mvcService.projectInfo.bomNumber.startsWith('60YV0JS1')
) {
  const bomNo = this._mvcService.projectInfo.bomNumber.toUpperCase().substring(0, 8);
  this._modalService.toggleVgaSpecialAlertModal(bomNo);
}
else if (
  this._mvcService.projectInfo.bomNumber.startsWith('60YV0NF2') ||
  this._mvcService.projectInfo.bomNumber.startsWith('60YV0PA2')
) {
  this._modalService.toggleVgaSpecialAlertModal('60YV0ID7');
}
```

這段邏輯的問題：純粹的字串比對規則，卻只能透過建構整個元件、模擬一堆依賴才能測試到。改善做法是抽成純函數：

```ts
// vga-special-alert-bom.util.ts —— 純函數，輸入輸出明確，零依賴
export function getVgaAlertBomNo(bomNumber: string): string | undefined {
  if (
    bomNumber.startsWith('60YV0ID7') ||
    bomNumber.startsWith('60YV0JS0') ||
    bomNumber.startsWith('60YV0JS1')
  ) {
    return bomNumber.toUpperCase().substring(0, 8);
  }

  if (
    bomNumber.startsWith('60YV0NF2') ||
    bomNumber.startsWith('60YV0PA2')
  ) {
    return '60YV0ID7';
  }

  return undefined;
}
```

```ts
// app.component.ts —— 元件只剩下「呼叫 + 執行副作用」
const vgaAlertBomNo = getVgaAlertBomNo(this._mvcService.projectInfo.bomNumber);
if (vgaAlertBomNo) {
  this._modalService.toggleVgaSpecialAlertModal(vgaAlertBomNo);
}
```

`getVgaAlertBomNo` 現在可以用 8 個測試案例（5 個比對分支 + 不比對 + 空字串 + 大小寫邊界）完全覆蓋，且測試執行時間是毫秒級，不需要 `TestBed`、不需要任何 mock。

同一份重構在本專案還做了 4 次，模式完全一致：

| 抽出的純函數 | 原本埋在哪裡 | 輸入 → 輸出 |
|---|---|---|
| `isInvalidTokenId()` | `alive()` 方法內的 if 判斷 | `tokenId` → `boolean` |
| `shouldShowUcsInfoPanel()` | `getApiData()` 內的 4 條件 OR 判斷 | UCS 資料物件 → `boolean` |
| `findSubmenuByProblemKey()` | `init()` 內的雙層 for 迴圈 + break flag | `(menus, problemkey)` → 匹配的 submenu 或 undefined |
| `decideVoltageCheckPointLayerSwitch()` | `init()` 內的 if/else if 層別判斷 | `(topCnt, btmCnt, currentLayer)` → 目標層別或 undefined |
| `decideChipsetRepairMouseoverAction()` | `mouseover` 事件監聽器內的巢狀 if/else | `(chipsetRepairs, partNumber, openLogs, picLogs)` → 決策物件 |

### 新專案該怎麼做

- **看到 `if`/`switch` 且沒有副作用（不呼叫 service、不寫 DOM）的區塊，優先考慮抽成純函數**，命名慣例可用 `*.util.ts`，放在跟被抽出的來源檔案同一層或同一個功能資料夾。
- 純函數的檔案**不需要注入任何東西**，天然就是最好測試、最不會出錯的程式碼單位。
- 抽出時要注意**保留原始邊界行為**，不要順手「順便修正」看起來奇怪但實際上是刻意設計或已知風險極低的邏輯（例如本專案 `alive()` 裡兩處寫法不同的失效判斷，統一時要明確意識到、並告知團隊這是行為上的微調，而不是單純搬移）。

---

## 3. 決策邏輯與副作用要分開（尤其是牽涉到共享可變狀態時）

### 本專案的案例

Chipset Repair 的 mouseover 判斷邏輯原本是這樣：決定要做什麼、寫入 DOM 屬性、push 到記錄陣列、呼叫 modal，全部揉在同一段 if/else 裡：

```ts
// 修改前
if (chipsetItem.CpuTitle && this._chipsetRepairService.chipsetRepairOpenlogs.indexOf(chipsetItem.PartNo) === -1) {
  element.parentElement?.setAttribute('cpu-impedance-notice', 'true');
  this._chipsetRepairService.chipsetRepairOpenlogs.push(chipsetItem.PartNo);
  if (this._configService.isShowCpuImpedanceCheckNotice) {
    this._modalService.toggleChipsetRepairWindow({ ChipsetRepairData: chipsetItem, ShowContent: true });
  }
  break;
}
else if (chipsetItem.Information) {
  // ...類似的混合邏輯
}
```

改善做法是讓一個純函數**只負責回答「要做什麼」**，回傳一個 discriminated union 描述決策結果；實際的 DOM 寫入、陣列 push、service 呼叫則留在呼叫端執行：

```ts
export type ChipsetRepairMouseoverAction =
  | { type: 'notice'; chipsetItem: ChipsetRepair }
  | { type: 'info'; chipsetItem: ChipsetRepair; shouldTogglePicture: boolean }
  | { type: 'none' };

export function decideChipsetRepairMouseoverAction(
  chipsetRepairs: ChipsetRepair[],
  partNumber: string | null | undefined,
  openLogs: string[],
  picLogs: string[]
): ChipsetRepairMouseoverAction {
  // 只讀取傳入的陣列，不修改、不呼叫任何 service
  // ...
}
```

### 新專案該怎麼做

- 當一段邏輯同時「做決策」又「執行副作用」時，先問：如果把所有 service 呼叫拿掉，剩下的判斷邏輯還能用純輸入輸出描述嗎？如果可以，就把決策部分抽出、回傳一個結構化的結果（物件或 union type），呼叫端再依結果執行副作用。
- 這個模式的額外好處：決策邏輯的測試不需要 spy/mock 任何 service，只需要斷言回傳值。

---

## 4. Component 內直接操作原生 DOM

### 症狀

`app.component.ts` 的 `init()` 方法直接用 `document.querySelectorAll('circle').forEach(...)` 綁定 `contextmenu`/`mouseover`/`mouseout` 事件，`onResize()` 直接用 `document.getElementById(...)`、`element.setAttribute(...)` 操作 SVG 屬性。

### 為什麼不好

- 這繞過了 Angular 的變更偵測與渲染管線，框架不知道這些 DOM 節點被動態修改過。
- 混用原生 DOM API 跟 Angular 綁定語法，會讓同一個元件同時存在兩套「畫面更新」的心智模型，維護時容易搞混。
- 這類程式碼幾乎無法在不啟動真實瀏覽器 DOM 的情況下測試，也是本專案測試覆蓋率低落的原因之一。

### 新專案該怎麼做

- 優先用 Angular 的 `Renderer2`、`HostListener`、Structural/Attribute Directive 來處理 DOM 互動，而不是在元件方法裡手動 `querySelectorAll`。
- 如果某一群元素（例如本例的每個 `circle`）都需要同一組事件處理邏輯，這是建立一個專屬 **Directive**（例如 `appCircleInteraction`）的明確訊號——把 `contextmenu`/`mouseover`/`mouseout` 的處理邏輯全部搬進 Directive，套用在模板上，而不是用 JS 迴圈手動綁定。這同時能把好幾個目前塞在根元件的 service（`ContextMenuService`、`ToolTipService`、`VirtualGroupService`……）一起移出去。
- 如果真的無法避免直接操作 DOM（例如效能考量、SVG 動態繪製），至少把這些操作集中在一個明確命名的 service/helper 裡，不要散落在元件方法之間。

---

## 5. 龐大的非同步 Orchestration 邏輯塞進 `ngOnInit`

### 症狀

`ngOnInit()` 裡有一長串 `await` 呼叫鏈：畫圖 → 等待完成 → 取得 Tutorial Video 資料 → 呼叫 `init()` → 關閉 loading → `setTimeout` 等 repaint → 呼叫 `getApiData()`。這個順序本身是刻意設計的（註解裡明確寫了「確保畫圖完成才能繼續」），但全部寫在元件生命週期方法裡，難以單獨測試這條時序邏輯本身。

### 新專案該怎麼做

- App 啟動時的「多步驟、有嚴格順序要求」的非同步流程，建議整個搬進一個專門的 Service（例如 `AppBootstrapService.run()`），讓它回傳一個 `Promise`/`Observable`，元件只需要 `await this._bootstrapService.run()`。
- 這類 orchestration 邏輯改動風險本來就高（時序錯了很難立刻發現），**不建議在沒有測試網的狀況下貿然重新排列**；如果要做，先確保搬移前後呼叫順序、await 點完全一致，再逐步補強測試。

---

## 6. 移除/搬移建構子依賴前，一定要檢查循環依賴風險

### 本專案的真實教訓

專案裡的 `CommonService` 檔案開頭就寫著明確的設計意圖：

```ts
/**
 * 共用Service，存放共用function，避免Service間互相注入
 */
```

它注入了 `MvcService`、`MeasureSignalService`、`KnowledgeBaseService`、`PartsRankingService`、`TutorialVideoService`、`VirtualGroupService`、`UcsService` 共 7 個 service，目的就是**避免這些 service 之間互相注入形成循環**。

在檢討「哪個建構子依賴可以移除」時，一開始曾考慮把 `AppComponent.updateMenuVisibility()`（目前只呼叫 `CommonService.menuVisiableSetting()`）整段搬進 `NavBarService`。實際檢查依賴圖後發現：`VirtualGroupService`（`CommonService` 的依賴之一）本身就注入了 `NavBarService`。如果讓 `NavBarService` 反過來注入 `CommonService`，會直接形成：

```
NavBarService → CommonService → VirtualGroupService → NavBarService
```

這是一個循環依賴，Angular 的 DI 系統無法解析，執行時會出錯。

### 新專案該怎麼做

- **搬移某段邏輯到另一個 service 之前，先確認目標 service 的完整依賴鏈，不要只看它「現在」的依賴，也要往下追一層它的依賴的依賴。**
- 優先挑選「本身零依賴、或依賴極少」的 service 作為搬移/合併的起點——一個完全沒有建構子依賴的 service（本專案的 `StatusBarService` 就是這種情況：`constructor() {}`）不可能參與循環，是結構上最安全的選擇。
- 如果團隊已經發現某類 service 有循環依賴的傾向（就像本專案的 `CommonService` 存在的理由），**把這個決策原因寫成註解留下來**，避免未來的人（包含未來的自己）不知道背景就把邏輯搬回去、重新製造循環。
- 專案裡也可以看到另一種應對模式：`NavBarService`、`SvgService` 用 `Injector.get(X)` 做成 lazy getter，並註明「改為 lazy getter，避免循環依賴」。這是當兩個 service **真的需要互相知道對方**時的合理退路，但屬於「不得已才用」的手段，不該是預設做法——優先設計成單向依賴，只有在真的無法避免雙向關係時才用 lazy getter 打斷循環。

---

## 7. 測試覆蓋率與可測試性要從第一天就顧到，不要事後補

### 本專案的真實情況

在檢視這個專案時，101 個既有的 spec 檔案裡，**除了新補的商業邏輯測試外，其餘全部都只是 Angular CLI 產生的樣板**（`expect(service).toBeTruthy()`），而且大部分連編譯都通不過——因為 `tsconfig.spec.json` 的 `exclude` 設定跟主專案衝突，加上一堆 service/component 升級到 Angular standalone 後，樣板測試也沒有跟著更新（`declarations` 該改成 `imports`、建構子多了新參數但測試沒補參數）。等到真正要讓測試「能跑」，得先建一整套 `BASE_TEST_PROVIDERS`（模擬 `MvcService`、`TranslateService`、`HttpClient`、`ToastrService`、動畫 provider）才能讓最基本的「元件能不能被建立」測試通過。

這個狀態本身就是「元件設計違反可測試性」的直接證據：如果一個 service/component 光是要被建立起來，就需要一整套跨系統的假資料，代表它的依賴設計本來就有問題。

### 新專案該怎麼做

- **CLI 產生的樣板測試（`expect(x).toBeTruthy()`）沒有實際驗證價值，不要讓它長期停留在專案裡當作「測試覆蓋率」的假象。** 要麼補上真正的行為斷言，要麼在確認沒有維護價值時直接刪除——樣板測試如果長期沒人維護，遲早會變成升級時的阻礙（本專案就是活生生的例子）。
- **優先把「決策邏輯」抽成純函數並補測試**（見第 2、3 節），這是投入最少、回報最高的測試策略：不需要 `TestBed`、不需要 mock 任何東西。
- 如果元件/服務必須依賴一堆共同的底層服務（例如本專案的 `MvcService` 讀取 `window.mvcViewModel`），考慮從專案初期就建立一組共用的測試 provider/stub（類似本專案後來補的 `src/testing/service-stubs.ts`），不要等到要修測試時才發現每個 spec 都要各自兜一套。
- 升級 Angular 版本、把元件改成 standalone 時，**同步檢查並更新對應的 spec 檔案**，不要讓測試因為版本升級而默默失效。

---

## 8. 避免在 class field 初始化時就依賴建構子注入的值

### 症狀

```ts
barcodeSn = this._mvcService.indexViewModel.BarcodeSN;
ucsSn = this._mvcService.indexViewModel.UcsInfo?.sn;
```

這種寫法能正常運作，是因為 TypeScript 編譯後，建構子參數屬性（parameter properties）的賦值會在其他 class field 初始化之前執行——但這是一個**隱性的、依賴編譯器行為與宣告順序**的規則，肉眼從程式碼本身看不出「這樣寫為什麼是安全的」，未來如果調整 `tsconfig` 的 `useDefineForClassFields` 或欄位宣告順序，行為可能改變。

### 新專案該怎麼做

- 需要依賴建構子注入值來初始化的欄位，優先在建構子**主體內**明確賦值（`this.barcodeSn = this._mvcService.indexViewModel.BarcodeSN;`），讓執行順序一目了然，不要依賴 class field 初始化順序的隱性規則。

---

## 9. 不要在檔案裡留存大段被註解掉的程式碼

`app.component.ts` 的 `ngOnInit()` 尾端留有一整段被註解掉的 `MutationObserver` 邏輯（約 20 行）。這類「以防以後要用」的死程式碼，實務上幾乎不會有人重新啟用，只會造成閱讀干擾與誤判風險（新人可能誤以為這段邏輯還在運作，或花時間去理解一段根本不會執行的程式）。

**新專案該怎麼做**：需要保留歷史邏輯，交給 Git 歷史記錄；程式碼裡不需要的區塊直接刪除。

---

## 總結檢查清單

設計新 Angular 元件、或 Review 別人的 PR 時，可以用這份清單快速檢查：

- [ ] 這個元件的建構子依賴數量，是不是明顯超出「這個元件實際要做的事」所需？如果超過 8-10 個，該考慮拆出一個 orchestrator/bootstrap service 了。
- [ ] 元件方法裡有沒有超過一行、沒有副作用的 `if`/`switch` 判斷？有的話，能不能抽成純函數？
- [ ] 有沒有「決策」跟「副作用」混在同一段程式碼裡？能不能拆成「純函數回傳決策」+「呼叫端執行副作用」兩層？
- [ ] 元件裡有沒有直接操作原生 DOM（`document.querySelector`、`addEventListener`、`setAttribute`）？能不能改用 `Renderer2`、`HostListener`、或抽成 Directive？
- [ ] 新增/搬移一個 service 依賴之前，有沒有檢查過目標位置的完整依賴鏈，確認不會造成循環注入？
- [ ] 每個 service/component 產生時，是否同時補上「有實際驗證價值」的測試，而不是留一個空的 `toBeTruthy()` 樣板？
- [ ] 有沒有依賴建構子注入值的 class field 初始化寫法？是否該搬進建構子主體讓順序明確？
- [ ] 有沒有大段被註解掉、確定不會再用的程式碼該清掉？

---

## 附錄：本專案實際重構對照表

| 抽出項目 | 原始位置 | 新檔案 | 對應測試 |
|---|---|---|---|
| VGA bomNumber 判斷 | `app.component.ts` `init()` | `features/modal/vga-special-alert-modal/vga-special-alert-bom.util.ts` | 8 個測試案例 |
| Token 失效判斷 | `app.component.ts` `alive()` | `utils/utils.ts`（`isInvalidTokenId`） | 5 個測試案例 |
| UCS Info 顯示判斷 | `app.component.ts` `getApiData()` | `core/services/ucs-info-panel.util.ts` | 6 個測試案例 |
| problemkey 對應 submenu | `app.component.ts` `init()` | `core/services/find-submenu-by-problem-key.util.ts` | 5 個測試案例 |
| Voltage Checkpoint 層別切換 | `app.component.ts` `init()` | `core/services/control-panel/voltage-checkpoint-layer.util.ts` | 6 個測試案例 |
| Chipset Repair mouseover 決策 | `app.component.ts` `init()` mouseover handler | `core/services/chipset-repair-mouseover.util.ts` | 8 個測試案例 |
| StatusBarService 依賴移除 | `app.component.ts` 建構子 | 搬進 `core/services/svg.service.ts`（`toggleRemindNoticeIfAny`） | 2 個測試案例 |

這些都是實際落地的重構，可以直接參考程式碼與對應的 `.spec.ts` 當作範例。
