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
export interface CircleContextMenuEvent {
  event: MouseEvent;
  element: SVGCircleElement;
}

@Directive({
  selector: '[appCircleInteraction]', // ✅ 套用在 Angular 認得的 #viz 容器上
  standalone: true
})
export class CircleInteractionDirective {

  @Output() circleContextMenu = new EventEmitter<CircleContextMenuEvent>();

  @HostListener('contextmenu', ['$event'])
  onContextMenu(event: MouseEvent): void {
    const element = event.target as SVGCircleElement;
    if (element.tagName?.toLowerCase() !== 'circle') {
      return; // 事件冒泡上來，但不是點在 circle 上，忽略
    }

    event.preventDefault(); // 擋掉瀏覽器原生右鍵選單
    this.circleContextMenu.emit({ event, element });
  }

  @HostListener('mouseover', ['$event'])
  onMouseOver(event: MouseEvent): void {
    const element = event.target as SVGCircleElement;
    if (element.tagName?.toLowerCase() !== 'circle') {
      return;
    }
    // ...顯示 tooltip、chipset repair 判斷等邏輯
  }

  @HostListener('mouseout', ['$event'])
  onMouseOut(event: MouseEvent): void {
    const element = event.target as SVGCircleElement;
    if (element.tagName?.toLowerCase() !== 'circle') {
      return;
    }
    // ...隱藏 tooltip
  }
}
```

套用在模板上：

```html
<!-- pcb-svg.component.html -->
<div id="viz" appCircleInteraction (circleContextMenu)="circleContextMenu.emit($event)">
</div>
```

`#viz` 是模板裡寫死的 `<div>`，Angular 編譯期認得它，所以 `[appCircleInteraction]` 這個 Directive 可以正常套用；而使用者在動態插入的 `circle` 上按右鍵/移過/移開時，`contextmenu`/`mouseover`/`mouseout` 這三個事件會冒泡到 `#viz`，被 Directive 的 `@HostListener` 收到，再用 `event.target` 判斷「這次事件真正發生在哪個元素上」。

### 一個額外要處理的細節：顯示右鍵選單的 UI 在另一個元件裡

實際顯示右鍵選單的 PrimeNG `<p-contextMenu>` 元件，宣告在 `AppComponent` 的模板裡，不是持有 `#viz` 的 `PcbSvgComponent`。Directive 沒辦法直接碰到別的元件模板裡的東西，所以做法是：Directive 只負責「確認是 circle + 組裝資料」，<span style="color: red;">透過 `@Output() circleContextMenu` 把事件送出去；持有 `#viz` 的元件把這個事件原封不動往外轉發；最外層的 `AppComponent` 監聽這個轉發後的事件，在那裡才真正呼叫顯示選單的邏輯。這是 Angular 常見的「子元件事件逐層往上轉發」模式，跟事件冒泡是兩個不同層級的機制（一個是瀏覽器 DOM 事件冒泡，一個是 Angular 元件之間的 `@Output`/`@Input` 通訊），只是恰好都在解決類似的問題：「這個資訊在 A 產生，但要在 B 使用」。</span>

#### `@Output()` 的基本用法

`@Output()` 是 Angular 讓子層（元件或 Directive）「主動通知外層發生了什麼事、並附帶資料」的機制，一定要搭配 `EventEmitter` 使用：

```ts
// 子層（元件或 Directive）
export class ChildThing {
  @Output() somethingHappened = new EventEmitter<SomeType>();

  private notifyParent(payload: SomeType) {
    this.somethingHappened.emit(payload); // 送出事件跟資料
  }
}
```

外層模板監聽（圓括號 `()` 是 Angular 綁定「事件」的語法，對應方括號 `[]` 綁定「屬性」）：

```html
<app-child-thing (somethingHappened)="onSomethingHappened($event)"></app-child-thing>
```

`$event` 在這裡就是 `emit(payload)` 傳進去的那個 `payload`。

**跟 DOM 事件冒泡最大的不同**：`@Output()` 完全是 Angular 自己的機制，只在模板裡有寫父子關係綁定的元件/Directive 之間才會生效，**不會**像
瀏覽器原生事件那樣自動沿著元件樹一路往上傳——每一層都要自己手動用 `(eventName)="..."` 接住，再自己決定要不要往上再轉發一次。對照本篇案例的
完整鏈路：

1. `CircleInteractionDirective`（套在 `#viz` 上）：`this.circleContextMenu.emit({ event, element })`
2. `PcbSvgComponent`（持有 `#viz` 的元件）模板：`(circleContextMenu)="circleContextMenu.emit($event)"` —— 接住 Directive 的事件，透過自己*
*同名的** `@Output()` 原封不動再送出去一次
3. `AppComponent` 模板：`<app-pcb-svg (circleContextMenu)="onCircleContextMenu($event)"></app-pcb-svg>` —— 接住 `PcbSvgComponent` 轉發的事
件，在對應方法裡才真正呼叫 `this.cm?.show(event)` 顯

三層缺一個綁定，事件就傳不過去——這跟冒泡「不用手動接、自動往上跑」是完全相反的心智模型，這也是為什麼这篇特別把兩者放在一起比較。


### 這次修正額外帶來的好處

1. **監聽器數量從 O(n) 降到 O(1)**：原本是幫每一個 `circle`（可能上千個）各自掛 3 個監聽器，現在整張圖只有 3 個監聽器（掛在 `#viz` 上）。
2. **修好了一個隱藏的既有缺陷**：原本的寫法只在初始化那一刻幫「當時存在」的 circle 掛監聽器，如果之後切換層別重新繪圖，新畫出來的 circle 不會被重新綁定，這三個事件就永久失效了。改成事件代理後，新插入的 circle 不需要任何額外動作就能自動生效。
3. **元件的建構子依賴變少了**：因為這三個事件處理邏輯需要用到的幾個 service（tooltip、VG 高亮、chipset repair 相關）原本注入在根元件裡，現在整段邏輯搬進 Directive 之後，根元件完全不再需要這幾個依賴。

---

## 五、總結

- **事件冒泡**：子元素觸發的事件會沿 DOM 樹往上傳給每一層祖先，前提是這個事件本身會冒泡（多數滑鼠/鍵盤事件會，`mouseenter`/`mouseleave`/`focus`/`blur` 不會）。
- **事件代理**：利用冒泡機制，把監聽器集中掛在共同的祖先元素上，靠 `event.target` 判斷實際觸發來源，取代對每個元素各自綁定監聽器。
- **Angular Directive 的 `selector` 比對，只認 Angular 自己渲染出來的元素**；跟事件冒泡是完全獨立的兩套機制——`selector` 決定 Directive 能不能被放上去，冒泡決定放上去之後能收到哪些事件。
- 當要處理「一大群同類型元素的事件」時，先確認這些元素是不是 Angular 渲染的：
  - 是 → 可以直接把 `selector` 設成該元素，也可以視效能需求選擇用事件代理套在共同祖先上。
  - 不是（例如由第三方繪圖函式庫、Web Worker 動態插入的原生 DOM）→ **必須**用事件代理，套在一個 Angular 認得、且包含這些元素的穩定祖先節點上，這是唯一還能用宣告式 Directive 反應到這些事件的辦法。
