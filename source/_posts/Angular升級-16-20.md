---
title: Angular升級(16 => 20)注意事項
date: 2026-04-22 09:44:54
tags:
---
##### 近期幫專案的前端升級，由原本的Angular16升級到Angular20，把一些比較重大的變化記錄下來。
<!-- more -->
#
#
#
---
### Angular的編譯方式：webpack vs esbuild
* ##### v16以前：
    ##### 使用 <span style="color:red;">Webpack</span>打包，處理大型專案時解析速度較慢。開發時為了模組化，會寫幾百個 .ts、.scss、.html 檔案。如果瀏覽器要一個一個去抓，載入速度會慢到崩潰。Webpack會從 main.ts 開始，像拉毛線球一樣把所有相關聯的檔案找出來，並根據類型進行合併成單一檔案(ex: 所有js會被合併為main.js, css則會合併成style.css....etc)。最後透過 index.html 一次載入。<span style="color:red;">Webpack 的本質就是 「將複雜的開發原始碼，轉換為瀏覽器好消化的單一（或少數）靜態資源」。</span>

* ##### v17+：
    ##### Angular17開始，預設改用 <span style="color:red;">Vite + esbuild</span>。打包速度比 Webpack 快非常多。<span style="color:red;">esbuild跟webpack本質是在做一樣的事情，但esbuild因為底層程式碼是使用go來開發，執行是會轉譯成機器語言，並可多執行緒執行打包工作。執行時，速度會比webpack的javascript直譯語言，且單一執行緒執行還要快許多。</span>

##### webpack 與 esbuild 本質上都是模組打包器（Bundler），但在執行效率上有代差級的差異：
* ##### 語言層級：webpack 是由 JavaScript 撰寫，執行時依賴 Node.js 直譯（Interpreted）；而 esbuild 是使用 Go 語言 開發，執行前已編譯為 機器原生代碼（Native Machine Code），讓 CPU 能直接執行指令。
* ##### 執行架構：JavaScript 本質上受限於 單執行緒（Single-threaded） 運行，而 Go 語言天生具備強大的 併發（Concurrency） 能力，讓 esbuild 能充分利用多核心 CPU 多執行緒 平行處理打包任務。
##### 結論：這使得 esbuild 的處理速度比 webpack 快上 10 到 100 倍。這也是 Angular 17+ 為了追求極致開發體驗（DX），將預設引擎從 webpack 遷移至 esbuild 的主因。
#
#
#
---
### Angular的模組載入方式：NgModule vs Standalone
* ##### v15以前：
    ##### 「在 Angular 15 以前，所有的元件（Component）必須歸類在某個 NgModule 之中（預設為 AppModule）。NgModule 充當了元件的封閉邊界，開發者必須透過模組來控管元件的 import 與 export 以限制其使用範圍。<span style="color:red">這種架構下，所有的子模組最終都會被根模組（AppModule）匯入。這導致 Angular 在啟動時，main.ts 必須先掃描並加載整個龐大且具備複雜依賴關係的模組樹。即便某些元件在首頁並未用到，也會因為被包裹在模組中而被迫載入，這不僅增加了初始包（Bundle Size）的大小，也導致應用程式的啟動與首屏渲染時間變長。」</span>

    ##### 下圖為Angular 16的AppModule，可以看到在import的部分，import了所有的子模組，子模組中則包含了各個元件。以圖中的FeaturesModule來說，裡面就包含了37個元件。其實大部分的元件在一開始的首頁，是用不到的，但因為Angular的Module載入機制，導致所有元件在一開始時就會一起載入，因而拖慢了時間。
    {% asset_image appModule.jpg appModule %}

    ##### 下圖為Angular 16的main.ts，Angular啟動時透過bootstrapModule，載入整個AppModule，導致渲染時間過久。
    {% asset_image main.jpg main %}

* ##### v15+：
    ##### Angular15開始，新增了Standalone技術。Standalone 技術將 『相依性宣告』 的權限從模組（Module）下放到各個元件（Component）身上。元件不再被迫綑綁在龐大的模組樹中，而是能根據需求，自主決定要載入哪些必要的工具或模組。這種變革打破了以往 『一啟動就必須載入整棵模組樹』 的連帶責任，實現了更細粒度的 Tree-shaking（搖樹優化）。在啟動時，main.ts 僅需載入根組件及其直接相關的依賴，大幅減少了初始包體積（Bundle Size）並顯著提升了應用的首屏渲染速度。

    ##### 舉例來說，在以前如果在 AppModule 裡 import 了 ChartModule，即便你的首頁根本沒用到圖表，瀏覽器還是得先把那包笨重的 JS 下載完。有了Standalone技術後，首頁元件如果不寫 imports: [ChartModule]，打包工具（esbuild）就會聰明地把它從首頁的產物中剔除。

    ##### 建立專案時會問你要不要 Standalone，預設還是可以使用 AppModule全部載入的方式。

* ##### v17+：
    ##### 預設全面取消 NgModule，改採Standalone模式。

##### 專案升級至Angular 20，專案改用Standalone後，原本的NgModule就可以刪除了。在未改用Standalone前，專案主要有5大模組，分別為：
    ``` text
    AppModule
    └─ imports AppRoutingModule
    └─ imports CoreModule
        └─ imports SharedModule
    └─ imports FeaturesModule
        └─ imports SharedModule
    └─ imports SharedModule
    ```
##### SharedModule被CoreModule和FeaturesModule import，這些Module最後再被最上層的AppModule import。專案升級後，Standalone component 都直接在自己的 imports[] 裡引入各自需要的pipe/directive，完全不透過這些Module，因此這些Module確認都無使用後即可刪除。
##### 比較要注意的地方為：
  * ##### AppRoutingModule => 刪除後，由<span style="color: red;">app.route.ts跟app.config.ts</span>接管
  * ##### AppModule => 刪除後，進入點改由<span style="color: red;">maint.ts</span>進入
#
#
#
---
### Angular偵測UI變化的機制：zone.js vs Signal
* ##### v18以前：
    ##### 透過pollyfills安裝zone.js補釘來偵測UI的更新，Zone.js是一個Angular的底層套件，會<span style="color: red;">主動監聽所有的非同步事件，監聽到了非同步事件後，會向 Angular 發出信號，Angular會在從根元件(<app-root>)往下遍歷檢查(變更偵測循環 [Change Detection Cycle])整個元件樹來找到要進行UI異動的地方，並將更新反映到對應的HTML元素上，所以當元件樹的規模越大時，遍歷的這個行為就會耗費許多電腦效能。</span>以下是 Zone.js 能夠偵測到的主要 UI 更新事件分類：
    1. ##### DOM 事件監聽 (DOM Event Listeners)
        ##### 這是最常見的觸發來源。只要是透過 addEventListener 綁定的事件，Zone.js 都會介入：
        * ##### 滑鼠事件：click, dblclick, mousedown, mouseenter, mouseleave, mousemove, mouseup。
        * ##### 鍵盤事件：keydown, keypress, keyup。
        * ##### 表單事件：input, change, submit, focus, blur。
        * ##### 滾動與縮放：scroll, resize（這在處理 Canvas 響應式佈局時常被觸發）。
    2. ##### HTTP 請求與通訊 (Network & Communication)
        ##### 與後端的API串接：
        * ##### XMLHttpRequest (XHR)：傳統的 Ajax 請求。
        * ##### Fetch API：現代瀏覽器的非同步請求機制。
        * ##### WebSocket：當後端主動推播資料時，產生的事件回呼也會被監聽。 
    3. ##### 非同步事件
    ##### 除了 timeout 與 interval，還包含：
    * ##### Promise：所有 .then() 或 .catch() 的回呼執行完後，Zone.js 都會通知 Angular 進行檢查。
    * ##### Subscribe：RxJS 內部的非同步步操作。

* ##### v18+：    
    ##### Angular 18開始，改採Zoneless(去Zone)模式，改用Signal來追蹤UI的更新，Zone監聽可以把他想成是針對整個元件樹的<span style="color: red;">整個面</span>來監控，那Signal則是針對個別元件，採用<span style="color: red;">點對點</span>的方式來進行監控，讓整個監控範圍縮小至一個點，讓UI更新的反應可以更迅速，也因為不用再去對整個元件樹遍歷檢查，節省了許多的電腦效能的消耗。雖然採用了Zoneless，但也不是說就不再需要zone.js這個套件了，為了與舊功能相容，Zone.js還是有保留的必要，<span style="color: red;">但從Angular 18開始，可以在app.config.ts中使用provideExperimentalZonelessChangeDetection()來徹底關閉Zone.js，達到100%的Zoneless。</span>下面是一個簡易的Signal追蹤的範例：
    ``` TypeScript
    import { Component, signal, computed } from '@angular/core';

    @Component({
    selector: 'app-counter',
    standalone: true,
    template: `
        <div>
        <h2>當前計數：{{ count() }}</h2>
        <p>雙倍數值：{{ doubleCount() }}</p>
        
        <button (click)="increment()">增加 +1</button>
        <button (click)="reset()">重設</button>
        </div>
    `
    })
    export class CounterComponent {
    // 1. 定義一個可變的 Signal
    count = signal(0);

    // 2. 定義一個衍生 (Computed) Signal
    // 當 count 改變時，這裡會自動計算，且具有快取機制
    doubleCount = computed(() => this.count() * 2);

    increment() {
        // 3. 更新 Signal 的值
        this.count.update(value => value + 1);
    }

    reset() {
        // 直接設定新值
        this.count.set(0);
    }
    }
    ```
#
#
#
---
### 補充：升級時遇到的問題
* ##### Error: NG0908
    ##### 透過AI幫我把專案改成Standalone模式後，出現了一個怪問題。
    * ##### 本機開發環境：執行ng s後，在本機執行起來都是正常的。

    * ##### 遠端測試環境：但放到遠端測試機後，卻一直出現 NG0908 的錯誤。最後才知道，如果 Angular 專案是透過 .NET Framework MVC 的 index.cshtml 來啟動，而非透過 Angular 本身的 index.html，則會有以下問題：ng build --output-hashing=all 執行後，zone.js（Angular 變更偵測機制的核心依賴，必須在 Angular 啟動前載入）會被單獨打包成獨立的 polyfills-{hash}.js，而不是合併進 main.js 裡。Angular CLI 會自動修改輸出的 index.html，注入對應的 &lt;script&gt; 標籤。但 MVC 的 index.cshtml 是手動維護的，Angular CLI 不會自動更新它，導致 zone.js 從未被載入，Angular 啟動時找不到必要的 zone.js，進而拋出 NG0908。<span style="color: red;">反觀舊版（Angular 16 dev build 不加 --output-hashing）不會有此問題，是因為當時 zone.js 會直接打包進 main.js，index.cshtml 只要載入 main.js 即可，zone.js 自然隨之載入。 </span>

    * ##### 最後的解決方法為：在main.ts的第一行加上import 'zone.js'。這樣的話，不管是透過MVC的index.cshtml還是本機的index.html進入Angular專案首頁，一定可以確保zone.js在Angular啟動時在main.ts載入。最後再把angular.json裡面的polyfills陣列內容清除。
    ``` text
    import 'zone.js'
    ```
#
#
#
* ##### UI更新失效
    ##### 在專案更新後，部分 UI 狀態發生變動後，畫面卻沒有反應。確認專案並未進行以下設定：
    * ##### 全域 Zoneless：尚未在 app.config.ts 中使用
    * ##### 元件級 OnPush：該元件的裝飾器尚未被加上 changeDetection: ChangeDetectionStrategy.OnPush。
    ##### <span style="color: red;">確定沒把專案設定為Zoneless或是把元件設定成OnPush的情況下，那很有可能是第三方套件的內部行為所導致。把原本的subscribe訂閱改成使用Signal應該可以解決沒有反應的問題。</span>
#
#
#
* ##### 改成lazy loading後，出現Error: Cannot use import statement outside a module
    ##### 當使用了@defer，Angular 會產出許多獨立的 chunk-xxx.js 檔案，這些檔案之間是透過原生ESM(import/export) 來互相溝通的。因為我的專案後端是.Net Framework MVC架構，首頁為index.cshtml。<span style="color: red;">要在&lt;script&gt; 標籤加上type="module"才能理解並執行 ESM(ECMA Script Modules)的import/export語法</span>
    ``` text
    @Scripts.RenderFormat("<script src='{0}' type='module'></script>", "~/Areas/Viewer/bundles/AngularJs")
    ```
#
#
#
---
### END