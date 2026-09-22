---
title: 事件冒泡（Event Bubbling）與 Angular Directive 事件代理實戰
date: 2026-08-25 14:57:41
categories:
 - 前端開發
tags:
 - Angular
 - AI產出
---
<!-- # 事件冒泡（Event Bubbling）與 Angular Directive 事件代理實戰 -->

這篇整理兩件事：

1. 瀏覽器的事件冒泡機制是什麼、哪些事件會冒泡。
2. 在 Angular 專案裡，如何利用冒泡機制，把「大量同類型元素各自綁定事件」的做法，改成「套一個 Directive 在共同的父層元素上」，並用一個真實案例（PCB 圖上動態繪製的 `<circle>` 元素）完整走一遍。
<span style="color:red;">文章內容透過ai整理產出</span>
<!--more-->
---

## 一、什麼是事件冒泡

當使用者在某個 DOM 元素上觸發一個事件（例如點擊），瀏覽器預設會讓這個事件**沿著 DOM 樹，從觸發的元素開始，一路往上傳遞給它的每一層祖先元素**，直到 `document`。這個「往上傳遞」的過程就叫做**冒泡（bubbling）**。

```html
<div id="grandparent">
  <div id="parent">
    <button id="child">點我</button>
  </div>
</div>
```

```js
document.getElementById('grandparent').addEventListener('click', () => console.log('grandparent 收到了'));
document.getElementById('parent').addEventListener('click', () => console.log('parent 收到了'));
document.getElementById('child').addEventListener('click', () => console.log('child 收到了'));
```

點擊 `#child` 按鈕，執行順序會是：

```
child 收到了
parent 收到了
grandparent 收到了
```

`child` 自己先收到事件，事件接著「冒泡」往上傳給 `parent`，再傳給 `grandparent`——即使 `parent`、`grandparent` 本身完全沒有被點擊，它們一樣收到了通知。

DOM 事件的完整生命週期實際上有三個階段：**捕獲（capturing，從最外層往下傳到目標元素）→ 目標（target，事件實際發生的元素）→ 冒泡（bubbling，從目標元素往上傳回外層）**。預設情況下 `addEventListener` 監聽的是冒泡階段，這也是本篇要討論的重點。

### 哪些事件會冒泡，哪些不會

| 事件 | 會不會冒泡 | 備註 |
|---|---|---|
| `click` | 會 | |
| `mouseover` / `mouseout` | 會 | |
| `contextmenu` | 會 | 瀏覽器的右鍵選單事件，本篇案例會用到 |
| `mousedown` / `mouseup` | 會 | |
| `keydown` / `keyup` | 會 | |
| `mouseenter` / `mouseleave` | **不會** | 只在滑鼠真正進入/離開該元素本身時觸發一次，不理會子元素邊界 |
| `focus` / `blur` | **不會** | 有對應會冒泡的版本：`focusin` / `focusout` |

**這件事很重要**：如果原本用的是 `mouseenter`/`mouseleave`/`focus`/`blur` 這類不冒泡的事件，「套在父層監聽」這個技巧完全不成立，父層永遠收不到子元素觸發的事件。使用這個技巧前，一定要先確認手上的事件種類。

---

## 二、為什麼冒泡機制重要：事件代理（Event Delegation）

假設一個頁面上有 1000 個同類型的元素（例如 1000 個 `<circle>`），每個都需要 `mouseover` 事件處理邏輯。

**傳統做法**：對每個元素各自呼叫 `addEventListener`。

```js
document.querySelectorAll('circle').forEach(circle => {
  circle.addEventListener('mouseover', handleMouseOver);
});
```

這樣會建立 1000 個監聽器，而且如果之後又動態新增了新的 `circle`，新元素不會自動套用這個監聽器，需要重新跑一次這段程式碼。

**事件代理做法**：利用冒泡機制，只在一個**共同的祖先元素**上掛一個監聽器，事件發生時透過 `event.target` 判斷實際是哪個元素觸發的。

```js
document.getElementById('container').addEventListener('mouseover', (event) => {
  const target = event.target;
  if (target.tagName.toLowerCase() !== 'circle') {
    return; // 不是點在 circle 上，忽略
  }
  handleMouseOver(target);
});
```

好處：

- **監聽器數量從 O(n) 降到 O(1)**——不管畫面上有 10 個還是 10000 個 `circle`，永遠只有 1 個監聽器。
- **動態新增的元素自動生效**——因為監聽器根本不是掛在個別元素上，新增的 `circle` 只要還在這個容器裡面，冒泡事件自然會傳到容器上，不需要重新綁定。

---

## 三、Angular 裡怎麼用：Directive + 事件代理

Angular 提供 `@Directive` + `@HostListener` 來取代手動的 `addEventListener`/`removeEventListener`，讓框架自動管理監聽器的掛載跟解除。但這裡有一個容易搞混的地方，值得特別拆開來看。

### `selector` 比對跟事件冒泡是兩個完全獨立的機制

- **`@Directive({ selector: ... })` 的比對**，是 Angular **編譯模板檔案的時候**，靜態分析 `.html` 原始碼裡有沒有符合這個 selector 的標籤，然後在渲染這個模板時把 Directive 實例接上去。**只有 Angular 自己編譯、自己渲染出來的元素**才會被比對到——如果一個 DOM 節點是用 `document.createElement`/`appendChild` 這種瀏覽器原生 API 在執行期手動插入的，不管它的 tag/class/id 多麼符合某個 selector 語法，Angular 都不會幫它套用任何 Directive，因為 Angular 從一開始就不知道這個節點存在。
- **事件冒泡**是瀏覽器的 DOM 樹層級行為，跟一個節點是不是 Angular 建立的**完全無關**。只要事件本身會冒泡（見上表），子元素觸發的事件就會一路往上傳到任何一層祖先——不管這個子元素是誰建立的。

這代表：**Directive「能不能掛上去」取決於目標元素是不是 Angular 渲染的；但 Directive 掛上去之後，「能收到什麼事件」取決於瀏覽器的冒泡機制，跟 Angular 完全無關。** 只要把 Directive 套在一個 Angular 認得、且是目標元素們的共同祖先的節點上，就能同時滿足這兩個條件。

---

## 四、實戰案例：PCB 圖上 circle 元素的事件代理

### 背景

專案裡有一頁會畫出整張 PCB 電路圖的 SVG，圖上大量的 `<circle>` 元素（可能上千個）代表元件的接腳，需要處理右鍵選單（`contextmenu`）、滑鼠移過（`mouseover`）、滑鼠移開（`mouseout`）三種事件（顯示 tooltip、chipset repair 提示、VG 高亮等功能）。

這些 `<circle>` 元素**不是寫在任何 Angular 模板（`.html`）裡的**，而是由 Web Worker 畫完 PCB 圖後，透過一個繪圖 service 用 `document.createElementNS(...)` + `appendChild(...)` 手動插入到頁面的 SVG 結構裡，插入的目標容器是一個 Angular 元件模板裡靜態寫著的 `<div id="viz"></div>`。

### 第一直覺（錯誤）：`selector: 'circle'`

```ts
@Directive({
  selector: 'circle', // ❌ 永遠不會生效
  standalone: true
})
export class CircleInteractionDirective { ... }
```

因為 `circle` 元素不是 Angular 渲染出來的，Angular 編譯期根本沒看過這個標籤出現在任何模板裡，這個 Directive 永遠不會被實例化——是一段死程式碼。

### 正確做法：套在 `#viz` 上，用事件代理判斷 `event.target`

```ts
// isCircleElement 是一個 type guard，Directive 跟 AppComponent 共用同一份判斷邏輯，
// 不用各自寫一份 tagName 比對。
export function isCircleElement(target: EventTarget | null): target is SVGCircleElement {
  return (target as Element | null)?.tagName?.toLowerCase() === 'circle';
}

@Directive({
  selector: '[appCircleInteraction]', // ✅ 套用在 Angular 認得的 #viz 容器上
  standalone: true
})
export class CircleInteractionDirective {

  @HostListener('mouseover', ['$event'])
  onMouseOver(event: MouseEvent): void {
    if (!isCircleElement(event.target)) {
      return; // 事件冒泡上來，但不是點在 circle 上，忽略
    }
    const element = event.target;
    // ...顯示 tooltip、chipset repair 判斷等邏輯
  }

  @HostListener('mouseout', ['$event'])
  onMouseOut(event: MouseEvent): void {
    if (!isCircleElement(event.target)) {
      return;
    }
    // ...隱藏 tooltip
  }
}
```

套用在模板上：

```html
<!-- pcb-svg.component.html -->
<div id="viz" appCircleInteraction>
</div>
```

`#viz` 是模板裡寫死的 `<div>`，Angular 編譯期認得它，所以 `[appCircleInteraction]` 這個 Directive 可以正常套用；而使用者在動態插入的 `circle` 上移過/移開時，`mouseover`/`mouseout` 這兩個事件會冒泡到 `#viz`，被 Directive 的 `@HostListener` 收到，再用 `event.target` 判斷「這次事件真正發生在哪個元素上」。

### `contextmenu` 為什麼沒有跟 mouseover/mouseout 放在一起處理

一開始的版本把 `contextmenu` 也塞進了這個 Directive，判斷完是不是 circle 後，透過 `@Output` 一路轉發（Directive → `PcbSvgComponent` 轉發一次 → `AppComponent` 才真正處理），理由是「顯示右鍵選單的 `<p-contextMenu>` 元件宣告在 `AppComponent` 模板裡，Directive 碰不到」。

但重新檢視後發現：`contextmenu` 跟另外兩個事件不一樣的地方是，它需要的東西（`ContextMenuService`、`<p-contextMenu>` 的元件參照、選單資料）**本來就已經是 `AppComponent` 既有的東西**，不是新增依賴。既然如此，根本不需要透過元件轉發——**`<app-pcb-svg>` 這個標籤本身就寫在 `app.component.html` 裡，是 `AppComponent` 渲染出來的真實 DOM 節點，範圍剛好完整包住 `#viz`**，可以直接在這個標籤上接原生的 `contextmenu` 事件，跳過 Directive、也跳過任何 `@Output` 轉發：

```html
<!-- app.component.html -->
<app-pcb-svg (contextmenu)="onCircleContextMenu($event)"></app-pcb-svg>
```

Angular 的 `(eventName)="..."` 語法，如果找不到同名的 `@Output`，會自動當成綁定該元素的**原生 DOM 事件**——這裡沒有任何 Directive 或元件宣告 `contextmenu` 這個 `@Output`，所以 Angular 直接幫我們掛上原生 DOM 事件監聽器。

```ts
// app.component.ts
onCircleContextMenu(event: MouseEvent): void {
  if (!isCircleElement(event.target)) {
    return;
  }
  const element = event.target;

  event.preventDefault(); // 擋掉瀏覽器原生右鍵選單
  this._contextMenuService.getContextContent(element).then(res => this.menuItems = res);
  this.cm?.show(event);
}
```

**代價**：「這是不是 circle」的判斷邏輯，現在分別出現在 Directive（`mouseover`/`mouseout`）跟 `AppComponent`（`contextmenu`）兩個地方，用同一個 `isCircleElement()` type guard 共用，避免重複寫兩份判斷條件。換來的好處是完全不需要 `@Output`、不需要中間的元件轉發，鏈路從三層縮成一層。

**還有一個小細節**：Directive 仍然保留了一個很小的 `contextmenu` 監聽，但只做「隱藏 tooltip」這一件事——因為這是 hover 生命週期的一部分（跟 `mouseout` 隱藏 tooltip是同一類邏輯），而 `ToolTipService` 已經是 Directive 既有的依賴。這代表同一個 `contextmenu` 事件，實際上會**同時**被兩個獨立的監聽器收到：Directive 上的（負責隱藏 tooltip）跟 `AppComponent` 上的（負責顯示選單）——這正好示範了冒泡機制的另一個特性：**同一個事件可以在冒泡路徑上的多個祖先層級，各自被獨立的監聽器處理，互不影響、不需要互相知道對方存在。**

### 這次修正額外帶來的好處

1. **監聽器數量從 O(n) 降到 O(1)**：原本是幫每一個 `circle`（可能上千個）各自掛監聽器，現在整張圖只需要固定幾個監聽器（`#viz` 上 2 個 + `<app-pcb-svg>` 上 1 個）。
2. **修好了一個隱藏的既有缺陷**：原本的寫法只在初始化那一刻幫「當時存在」的 circle 掛監聽器，如果之後切換層別重新繪圖，新畫出來的 circle 不會被重新綁定，事件就永久失效了。改成事件代理後，新插入的 circle 不需要任何額外動作就能自動生效。
3. **元件的建構子依賴變少了**：`mouseover`/`mouseout` 需要的 `ToolTipService`、`VirtualGroupService`、`ChipsetRepairService` 原本注入在 `AppComponent` 裡，搬進 Directive 之後，`AppComponent` 完全不再需要這幾個依賴；`contextmenu` 需要的 `ContextMenuService` 本來就在 `AppComponent` 身上，不多也不少。

---

## 五、總結

- **事件冒泡**：子元素觸發的事件會沿 DOM 樹往上傳給每一層祖先，前提是這個事件本身會冒泡（多數滑鼠/鍵盤事件會，`mouseenter`/`mouseleave`/`focus`/`blur` 不會）。
- **事件代理**：利用冒泡機制，把監聽器集中掛在共同的祖先元素上，靠 `event.target` 判斷實際觸發來源，取代對每個元素各自綁定監聽器。
- **Angular Directive 的 `selector` 比對，只認 Angular 自己渲染出來的元素**；跟事件冒泡是完全獨立的兩套機制——`selector` 決定 Directive 能不能被放上去，冒泡決定放上去之後能收到哪些事件。
- 當要處理「一大群同類型元素的事件」時，先確認這些元素是不是 Angular 渲染的：
  - 是 → 可以直接把 `selector` 設成該元素，也可以視效能需求選擇用事件代理套在共同祖先上。
  - 不是（例如由第三方繪圖函式庫、Web Worker 動態插入的原生 DOM）→ **必須**用事件代理，套在一個 Angular 認得、且包含這些元素的穩定祖先節點上，這是唯一還能用宣告式 Directive 反應到這些事件的辦法。
- **同一個事件，可以在冒泡路徑上的多個祖先層級，各自被獨立的監聽器處理**，不需要集中在一個地方。如果一組相關事件裡，其中一種的後續處理剛好已經是另一個元件的既有職責跟依賴，直接讓那個元件用原生事件綁定接住就好，不用勉強把所有事件塞進同一個 Directive、再用 `@Output` 逐層轉發——先看「這段邏輯的資料跟依賴本來就在哪裡」，再決定監聽器放哪裡，比先寫好 Directive 再想辦法轉發資料更省事。
