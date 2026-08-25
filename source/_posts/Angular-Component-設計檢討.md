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
- 如果某個 service 只在元件裡被使用一次、且用途單一，優先考慮這個依賴是否該屬於另一個「已經被注入」的 service，而不是讓元件直接持有它（詳見第 7 節「移除依賴前要檢查循環風險」）。

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

## 3. `*.util.ts`（純函數）跟 `*.service.ts`（Angular Service）的界線

抽邏輯出來時，另一個常見的疑問是：這段邏輯該放進 `*.util.ts` 還是變成某個 `*.service.ts` 的方法？兩者最核心的差異是「**有沒有狀態、要不要跟 Angular DI 系統打交道**」。

### 核心差異

|  | `*.util.ts`（純函數） | `*.service.ts`（Angular Service） |
|---|---|---|
| 有沒有 `@Injectable()` | 沒有，就是普通 TS 模組 | 有，才能被 Angular DI 容器管理 |
| 呼叫方式 | 直接 `import` 函數來用 | 要透過建構子注入或 `inject()` |
| 有沒有狀態 | **沒有**，每次呼叫都是全新計算，不記得上次呼叫發生過什麼 | **可以有**，而且通常就是為了「記住東西」才存在 |
| 有沒有副作用 | **不該有**（不呼叫 HTTP、不寫 DOM、不 emit 事件） | 通常就是負責副作用（API 呼叫、RxJS Subject、寫 localStorage） |
| 能不能在測試裡替換掉 | 不能，永遠是那個實作 | 能，靠 DI 的 `{ provide: X, useValue: mock }` 覆蓋——這也是本專案能把 `MvcService`、`TranslateService` 換成假物件的原因 |
| 生命週期 | 沒有，呼叫完就結束 | `providedIn: 'root'` 是整個 app 唯一一個實例（singleton），活得跟 app 一樣久 |

### 本專案的對照

- `getVgaAlertBomNo(bomNumber: string)` 是 util，因為它純粹是「給一個字串，回傳另一個字串或 undefined」，沒有任何依賴、沒有記憶、不需要被 mock。
- `SvgService.toggleRemindNoticeIfAny(remindNotice)` <span style="color: red;">做成 Service 的方法而不是 util，因為它**需要呼叫另一個被注入的 `StatusBarService`**（`this._statusBarService.toggleRemindNotice(...)`）——邏輯只要開始「跟外面的世界互動」（呼叫別的 service、發 HTTP、寫 DOM、emit 事件通知別的元件），就不再是純函數，天生就該是 Service 的職責。</span>
- `ChipsetRepairService.chipsetRepairOpenlogs`（`string[]`）也只能活在 Service 裡，因為它要**跨多次滑鼠事件記住「這個 partNumber 已經顯示過提示了」**——util 函數每次呼叫都是獨立的，沒有地方能存這種跨呼叫的狀態。

### 判斷準則

1. 這段邏輯要不要跟其他 service/API/DOM 互動？要 → Service。不要，純輸入輸出 → util。
2. 這段邏輯要不要記住上次呼叫的結果？要記住 → Service（狀態要有地方住）。不需要 → util。
3. 測試時需不需要把這段邏輯本身換成假的？如果連它都要被別人 mock 掉，它就該是能被 DI 覆蓋的 Service，不是 import 進來就跑死的 util 函數。

純函數不是一輩子都是純函數——如果之後某個 util 函數開始需要讀取某個 service 的內部狀態，而不是單純接收參數，這就是它該「升級」變成某個 Service 方法的訊號，不該硬留在 util 裡強迫呼叫端多傳一堆參數進去。

---

## 4. 決策邏輯與副作用要分開（尤其是牽涉到共享可變狀態時）

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

## 5. Component 內直接操作原生 DOM

### 症狀

`app.component.ts` 的 `init()` 方法直接用 `document.querySelectorAll('circle').forEach(...)` 綁定 `contextmenu`/`mouseover`/`mouseout` 事件，`onResize()` 直接用 `document.getElementById(...)`、`element.setAttribute(...)` 操作 SVG 屬性。

### 為什麼不好

- 這繞過了 Angular 的變更偵測與渲染管線，框架不知道這些 DOM 節點被動態修改過。
- 混用原生 DOM API 跟 Angular 綁定語法，會讓同一個元件同時存在兩套「畫面更新」的心智模型，維護時容易搞混。
- 這類程式碼幾乎無法在不啟動真實瀏覽器 DOM 的情況下測試，也是本專案測試覆蓋率低落的原因之一。

### 先建立概念：Directive 是什麼

Angular 裡有三種「東西」：

- **Component**：有自己的模板（HTML），例如 `<app-nav-bar>`。
- **Structural Directive**：控制「要不要畫」、「畫幾次」，例如 `*ngIf`、`*ngFor`（前面那個 `*` 就是 structural directive 的標記）。
- **Attribute Directive**：**不會**畫出新的 HTML，只是「附加行為」在已經存在的元素上——這是本節討論的重點。

寫法大概是這樣：

```ts
@Directive({
  selector: '[appHighlight]',   // 套用在有 appHighlight 屬性的元素上
  standalone: true
})
export class HighlightDirective {
  constructor(private el: ElementRef) {}

  @HostListener('mouseenter')
  onMouseEnter() {
    this.el.nativeElement.style.backgroundColor = 'yellow';
  }

  @HostListener('mouseleave')
  onMouseLeave() {
    this.el.nativeElement.style.backgroundColor = '';
  }
}
```

```html
<div appHighlight>滑鼠移過來會變黃色</div>
```

**核心概念**：不用自己寫 `document.querySelector(...)` 再 `addEventListener(...)`，而是把「這個元素該有什麼行為」宣告成一個 class，套在模板上的元素身上，Angular 會在對的時機自動把這個 class 的邏輯接上去（元素出現時接上、消失時自動解除，不用自己管生命週期）。這就是「往 Directive 走」的意思：**把命令式的 DOM 操作，換成宣告式的「這個元素套用這個行為」**。

**重要限制**：Directive 的 `selector` 比對，只作用在**透過 Angular 自己的渲染流程建立的元素**上（模板裡寫出來的、`*ngFor` 產生的……）。如果一個 DOM 節點是用 `document.createElementNS`/`appendChild` 這種原生 API 手動插進頁面的，Angular 完全不知道它的存在，`selector` 再怎麼寫都不會套用上去。這個限制在下面兩個案例都很關鍵。

### 本專案的案例：`onResize()` 搬到它真正該屬於的 Service

`onResize()` 原本整段寫在 `AppComponent` 裡：`@HostListener('window:resize')` 觸發後，直接 `document.getElementById('viz')` 查 DOM、手動 `setAttribute` 調整 SVG 跟拖曳背景矩形的尺寸，最後才呼叫 `this._svgService.onResize()` 通知其他訂閱者。

調查後發現：`#viz` 底下的 `<svg>` 跟 `.svgBackground`（`<rect>`）都是 `D3Service` 在畫圖時用 `document.createElementNS` 動態建立的，不是模板裡的靜態元素；而 `SvgService` 本身已經持有 `wWidth`/`wHeight` 這兩個狀態、已經有 `onResize()` 這個通知用的方法，甚至在別的地方也已經直接查詢過同一份 DOM（`document.querySelector('#viz > svg > g')`）。換句話說，**這段 DOM 操作邏輯本來就該屬於 `SvgService`，只是被寫錯了地方**——不需要為此新建一個 Directive，重新查一次 DOM、還要跟 `SvgService` 的狀態同步，反而是繞遠路。

修正後，`SvgService.onResize()` 把量測寬高、寫入 SVG 屬性、更新內部狀態、發通知全部收在一起，`AppComponent` 只留下監聽視窗事件的殼：

```ts
// app.component.ts —— 元件只剩下「監聽事件 + 委派」
@HostListener('window:resize')
onResize(): void {
  this._svgService.onResize();
}
```

這個案例的重點是：**「元件不該直接操作 DOM」不代表答案永遠是「新建一個 Directive」**。先確認這些 DOM 節點的建立者跟既有狀態的擁有者是誰——如果已經有一個 service 同時掌握相關狀態、又已經在操作同一份 DOM，通常直接搬進那個 service，比另外發明一層 Directive 更貼近現有架構、風險也更低。

### 反面教材：circle 事件綁定不能直接寫 `selector: 'circle'`

`init()` 裡還有一段更大的區塊：`document.querySelectorAll('circle').forEach(element => { element.addEventListener('contextmenu', ...); element.addEventListener('mouseover', ...); element.addEventListener('mouseout', ...); })`，幫每一個 `circle` 各自掛 3 個事件監聽器。

第一直覺會覺得這是「建立一個 `CircleInteractionDirective`，`selector: 'circle'`，套用在模板上」的明確訊號——但這個直覺**踩到上面提到的那個限制**：這些 `circle` 元素是 Web Worker 畫完 PCB 圖後，經由 `SvgDrawMultiThreadService` 用 `document.createElementNS` + `appendChild` 手動插進 DOM 的，**不是 Angular 模板渲染出來的**。如果把 Directive 的 `selector` 設成 `circle`，Angular 永遠不會把它套用到這些手動插入的元素上——這段 Directive 會變成一段永遠不會被觸發的死程式碼。

**正確做法**：把 Directive 套在一個 Angular 真的認得、真的有渲染的**外層容器**上（例如 `PcbSvgComponent` 模板裡的 `<div id="viz">`），搭配「事件代理（event delegation）」——監聽外層容器的事件，靠瀏覽器原生的事件冒泡機制往上傳（冒泡是瀏覽器層級的行為，跟元素是不是 Angular 畫的無關），再在處理函式裡檢查 `event.target` 是不是 `circle`：

```ts
@Directive({
  selector: '#viz',   // 套在 Angular 認得的外層容器上，不是動態插入的 circle 本身
  standalone: true
})
export class CircleInteractionDirective {

  @HostListener('mouseover', ['$event'])
  onMouseOver(event: Event) {
    const target = event.target as Element;
    if (target.tagName.toLowerCase() !== 'circle') {
      return; // 不是點在 circle 上，忽略
    }
    // ...原本 mouseover 的邏輯
  }

  @HostListener('contextmenu', ['$event'])
  onContextMenu(event: Event) { /* 同樣先檢查 target */ }

  @HostListener('mouseout', ['$event'])
  onMouseOut(event: Event) { /* 同樣先檢查 target */ }
}
```

這樣修正後還有兩個額外好處：

1. 現在的寫法是幫**每一個** `circle`（PCB 圖可能有成千上萬個）各自掛 3 個監聽器；改成外層容器上只掛 3 個監聽器、靠事件冒泡判斷 target，數量從 O(n) 降到 O(1)，效能更好。
2. 現在的寫法只在 `init()` 執行的那一刻幫「當時存在」的 circle 掛監聽器——如果之後切換 layer 重新繪圖，新畫出來的 circle **不會**被重新掛上這三個事件（這是現有程式碼一個隱藏的潛在缺陷）。改成容器層級的事件代理後，新插入的 circle 自動就能被監聽到，不需要重新執行任何綁定邏輯。

### 新專案該怎麼做

- 優先用 Angular 的 `Renderer2`、`HostListener`、Structural/Attribute Directive 來處理 DOM 互動，而不是在元件方法裡手動 `querySelectorAll`。
- 如果某一群元素都需要同一組事件處理邏輯，這是考慮 Directive 的訊號——但**先確認這些元素是不是 Angular 自己渲染出來的**。是的話可以直接把 `selector` 設成該元素；如果是像本例這樣由其他機制（Web Worker、第三方繪圖函式庫）動態插入的原生 DOM，Directive 要套在一個 Angular 認得的**穩定外層容器**上，搭配事件代理（檢查 `event.target`）來處理，不能天真地把 `selector` 設成動態元素本身的標籤名稱。
- 如果目前已經有一個 service 動態建立/掌管這些 DOM 節點，並且已經持有相關聯的狀態（如上面 `onResize()` 的案例），優先把邏輯搬進那個 service，而不是預設一律往 Directive 走——Directive 適合「需要套用在多個元素上的可重用行為」，不是所有 DOM 操作的萬用解法。
- 如果真的無法避免直接操作 DOM（例如效能考量、SVG 動態繪製），至少把這些操作集中在一個明確命名的 service/helper 裡，不要散落在元件方法之間。

---

## 6. 龐大的非同步 Orchestration 邏輯塞進 `ngOnInit`

### 症狀

`ngOnInit()` 裡有一長串 `await` 呼叫鏈：畫圖 → 等待完成 → 取得 Tutorial Video 資料 → 呼叫 `init()` → 關閉 loading → `setTimeout` 等 repaint → 呼叫 `getApiData()`。這個順序本身是刻意設計的（註解裡明確寫了「確保畫圖完成才能繼續」），但全部寫在元件生命週期方法裡，難以單獨測試這條時序邏輯本身。

### 新專案該怎麼做

- App 啟動時的「多步驟、有嚴格順序要求」的非同步流程，建議整個搬進一個專門的 Service（例如 `AppBootstrapService.run()`），讓它回傳一個 `Promise`/`Observable`，元件只需要 `await this._bootstrapService.run()`。
- 這類 orchestration 邏輯改動風險本來就高（時序錯了很難立刻發現），**不建議在沒有測試網的狀況下貿然重新排列**；如果要做，先確保搬移前後呼叫順序、await 點完全一致，再逐步補強測試。

---

## 7. 移除/搬移建構子依賴前，一定要檢查循環依賴風險

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

### 反面教材：循環依賴安全，不代表放的位置是對的

上面提到「`StatusBarService` 零依賴，是結構上最安全的搬移起點」——這句話本身沒錯，但實際操作時只顧到這一個條件，忽略了另一個同樣重要的問題：**這段邏輯放進目標 service 之後，是否符合目標 service 的職責範圍？**

實際發生的狀況：為了讓 `AppComponent` 少一個建構子依賴，把 `toggleRemindNoticeIfAny()`（判斷要不要顯示 status bar 提醒公告）搬進了 `SvgService`。這個搬移**沒有任何循環依賴風險**（`StatusBarService` 零依賴），但「顯示提醒公告」跟 `SvgService` 的職責（SVG 繪圖）完全無關——結果是把 `AppComponent` 的 SRP（單一職責）問題，換了個地方、換小一號規模再犯一次，而不是真正解決它。

正確的做法：這段邏輯理應收在 `StatusBarService` 自己身上（「要不要顯示」的判斷跟「怎麼顯示」的機制本來就該在一起），即使這代表 `AppComponent` 的建構子依賴數量沒有真的減少：

```ts
// StatusBarService —— 判斷跟顯示機制收在一起，呼叫端不用重複寫防呆判斷
toggleRemindNoticeIfAny(remindNotice: string): void
{
  if (remindNotice && remindNotice.length > 0)
  {
    this.toggleRemindNotice(remindNotice);
  }
}
```

**新專案該怎麼做（補充）**：

- 搬移一段邏輯前，至少要問兩個獨立的問題：(1) 會不會造成循環依賴？(2) 目標位置在職責上說得通嗎？**兩個都要成立，缺一不可**——第一個只是「不會炸」的必要條件，不是「放得對」的充分條件。
- 如果為了降低某個元件的依賴數量，唯一能找到的「安全」去處是一個職責不相關的 service，這通常代表**這段邏輯本來就該留在原地**，或是該去建立一個新的、職責名稱對得上的 service/orchestrator，而不是硬塞進湊巧安全的既有 service。減少依賴數量不是目的，只是「職責分配正確」帶來的自然結果——反過來為了湊數字而犧牲職責邊界，是本末倒置。

---

## 8. 測試覆蓋率與可測試性要從第一天就顧到，不要事後補

### 本專案的真實情況

在檢視這個專案時，101 個既有的 spec 檔案裡，**除了新補的商業邏輯測試外，其餘全部都只是 Angular CLI 產生的樣板**（`expect(service).toBeTruthy()`），而且大部分連編譯都通不過——因為 `tsconfig.spec.json` 的 `exclude` 設定跟主專案衝突，加上一堆 service/component 升級到 Angular standalone 後，樣板測試也沒有跟著更新（`declarations` 該改成 `imports`、建構子多了新參數但測試沒補參數）。等到真正要讓測試「能跑」，得先建一整套 `BASE_TEST_PROVIDERS`（模擬 `MvcService`、`TranslateService`、`HttpClient`、`ToastrService`、動畫 provider）才能讓最基本的「元件能不能被建立」測試通過。

這個狀態本身就是「元件設計違反可測試性」的直接證據：如果一個 service/component 光是要被建立起來，就需要一整套跨系統的假資料，代表它的依賴設計本來就有問題。

### 新專案該怎麼做

- **CLI 產生的樣板測試（`expect(x).toBeTruthy()`）沒有實際驗證價值，不要讓它長期停留在專案裡當作「測試覆蓋率」的假象。** 要麼補上真正的行為斷言，要麼在確認沒有維護價值時直接刪除——樣板測試如果長期沒人維護，遲早會變成升級時的阻礙（本專案就是活生生的例子）。
- **優先把「決策邏輯」抽成純函數並補測試**（見第 2、4 節），這是投入最少、回報最高的測試策略：不需要 `TestBed`、不需要 mock 任何東西。
- 如果元件/服務必須依賴一堆共同的底層服務（例如本專案的 `MvcService` 讀取 `window.mvcViewModel`），考慮從專案初期就建立一組共用的測試 provider/stub（類似本專案後來補的 `src/testing/service-stubs.ts`），不要等到要修測試時才發現每個 spec 都要各自兜一套。
- 升級 Angular 版本、把元件改成 standalone 時，**同步檢查並更新對應的 spec 檔案**，不要讓測試因為版本升級而默默失效。

---

## 9. 避免在 class field 初始化時就依賴建構子注入的值

### 症狀

```ts
barcodeSn = this._mvcService.indexViewModel.BarcodeSN;
ucsSn = this._mvcService.indexViewModel.UcsInfo?.sn;
```

這種寫法能正常運作，是因為 TypeScript 編譯後，建構子參數屬性（parameter properties）的賦值會在其他 class field 初始化之前執行——但這是一個**隱性的、依賴編譯器行為與宣告順序**的規則，肉眼從程式碼本身看不出「這樣寫為什麼是安全的」，未來如果調整 `tsconfig` 的 `useDefineForClassFields` 或欄位宣告順序，行為可能改變。

### 新專案該怎麼做

- 需要依賴建構子注入值來初始化的欄位，優先在建構子**主體內**明確賦值（`this.barcodeSn = this._mvcService.indexViewModel.BarcodeSN;`），讓執行順序一目了然，不要依賴 class field 初始化順序的隱性規則。

---

## 10. 不要在檔案裡留存大段被註解掉的程式碼

`app.component.ts` 的 `ngOnInit()` 尾端留有一整段被註解掉的 `MutationObserver` 邏輯（約 20 行）。這類「以防以後要用」的死程式碼，實務上幾乎不會有人重新啟用，只會造成閱讀干擾與誤判風險（新人可能誤以為這段邏輯還在運作，或花時間去理解一段根本不會執行的程式）。

**新專案該怎麼做**：需要保留歷史邏輯，交給 Git 歷史記錄；程式碼裡不需要的區塊直接刪除。

---

## 總結檢查清單

設計新 Angular 元件、或 Review 別人的 PR 時，可以用這份清單快速檢查：

- [ ] 這個元件的建構子依賴數量，是不是明顯超出「這個元件實際要做的事」所需？如果超過 8-10 個，該考慮拆出一個 orchestrator/bootstrap service 了。
- [ ] 元件方法裡有沒有超過一行、沒有副作用的 `if`/`switch` 判斷？有的話，能不能抽成純函數？
- [ ] 新抽出的邏輯，是該放進 `*.util.ts`（無狀態、不需要 DI），還是該是某個 `*.service.ts` 的方法（需要跟其他 service 互動、需要記住狀態、需要在測試裡被 mock）？
- [ ] 有沒有「決策」跟「副作用」混在同一段程式碼裡？能不能拆成「純函數回傳決策」+「呼叫端執行副作用」兩層？
- [ ] 元件裡有沒有直接操作原生 DOM（`document.querySelector`、`addEventListener`、`setAttribute`）？能不能改用 `Renderer2`、`HostListener`、或抽成 Directive？
- [ ] 新增/搬移一個 service 依賴之前，有沒有檢查過目標位置的完整依賴鏈，確認不會造成循環注入？**而且**目標位置的職責範圍跟這段邏輯對得上（循環安全跟放對地方是兩個獨立條件，都要成立）？
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
| Remind Notice 顯示判斷（`toggleRemindNoticeIfAny`） | `app.component.ts` `init()` | `core/services/status-bar.service.ts`（原本誤搬進 `svg.service.ts`，後修正，見第 7 節「反面教材」） | 2 個測試案例 |
| `onResize()` DOM 操作 | `app.component.ts` `onResize()` | `core/services/svg.service.ts`（`onResize()`），元件只留 `@HostListener` 委派 | 4 個測試案例 |

這些都是實際落地的重構，可以直接參考程式碼與對應的 `.spec.ts` 當作範例。**其中 Remind Notice 那一列本身就是一個真實的反例：第一次搬移只檢查了循環依賴風險，沒檢查職責歸屬，後來被抓出來修正——保留這段記錄，比只呈現「成功」的重構更有參考價值。**
