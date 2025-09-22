---
title: Webpack打包(以Chrome extension為範例)
date: 2025-09-08 16:05:49
categories:
 - 前端開發
tags:
 - Webpack
---
##### 先前開發Angular專案時，使用的是TypeScript。覺得使用 import 和 export 語法來模組化程式碼，讓可讀性變好這種方式很不錯。最近開發Chrome extension時，使用的是Js ES5(Vanilla Js)，就是只使用內建功能的原生Js來開發。
##### 後來問了一下ChatGPT後才知道，Js ES6開始，就可以使用import. export語法來把程式模組化，但最後需要使用 Bundler (例如 Webpack、Parcel、Rollup) 將這些模組打包成瀏覽器可以理解的格式，開發完的Chrome套件才能在瀏覽器上面正常執行。
##### 除了可以使用import. export之外，把Js打包後還有下列好處：
* ##### 使用 import 和 export 進行模組化開發：
* ##### 使用 Node.js (部分)模組：
    * ##### 沒有 Webpack： 你無法直接在 Chrome 擴充套件中使用 Node.js 模組，因為瀏覽器環境缺少 Node.js 的特定 API (例如 fs、http)。
    * ##### 使用 Webpack： Webpack 可以讓你使用部分不依賴於特定 Node.js 環境的純 JavaScript 模組。 雖然不能使用所有 Node.js 模組，但仍然可以利用許多有用的工具，例如 axios、lodash、moment 等。(*經實測，要處理圖片的Jimp套件就無法使用. 但原生js辨識圖像的套件Tesseract就可以使用)
* ##### 程式碼壓縮和最佳化：
    * ##### 你需要手動使用其他工具來壓縮和最佳化你的程式碼。
    * ##### Webpack 可以在生產模式下自動壓縮和最佳化你的程式碼，減少檔案大小，提高載入速度。
* ##### 程式碼分割 (Code Splitting)：
    * ##### 沒有 Webpack： 所有程式碼都會被打包到一個或幾個檔案中，導致初始載入時間較長。
    * ##### 使用 Webpack： 可以使用程式碼分割技術將程式碼分割成多個 chunk，只有在需要時才載入，提高初始載入速度。
##### 我只列出了跟我比較有相關，對我來說可以體感的有實質幫助的幾點， 下面為我的prompt。
``` text
簡單解釋chrome套件專案安裝了webpack後可以做哪些沒安裝前不能做的事
```
#
#
#
#
---
### Webpack安裝
#
#
#
#
##### 在 Chrome 擴充套件專案的根目錄中，執行以下指令，建立一個 package.json 檔案，用於管理專案的依賴。
```bash
npm init -y
```
#
#
#
#
##### 執行完畢後，可以看到方案總管多了package.json
{% asset_image package_json.jpg package_json %}
#
#
#
#
##### 執行以下指令安裝 Webpack、webpack-cli 和其他必要的 loader：
```bash
npm install webpack webpack-cli --save-dev
```
#
#
#
#
##### 如果之前都沒有安裝過Node.js套件的話，在方案總管會多出一個node_modules的資料夾，裡面會安裝很多webpack相關的依賴
{% asset_image node_modules.jpg node_modules %}
#
#
#
#
##### 可以把node_modules這個資料夾加入到.gitignore檔案裡面，這些東西不需要push到git上面，有需要時再npm install重新安裝即可
```plaintext
# Node.js 依賴
node_modules/

# 環境變數檔案
.env
.env.local
.env.*.local

# 日誌檔案
npm-debug.log*
yarn-debug.log*
yarn-error.log*
pnpm-debug.log*

# 建置輸出
dist/
build/
.cache/
.next/
out/

# 系統檔案
.DS_Store
Thumbs.db

# IDE 設定
.vscode/
.idea/

# 測試覆蓋率
coverage/

# 鎖檔（視需求可排除，也可保留）
# package-lock.json
# yarn.lock
# pnpm-lock.yaml
```
#
#
#
#
##### 建立 Webpack 設定文件(webpack.config.js)
```javascript
const path = require('path');

module.exports = {
  entry: {
    background: './background.js', // 背景腳本的入口點
    manifest: './manifest.js', // 彈出視窗腳本的入口點 (如果有的話)
  },
  output: {
    filename: '[name].js', // 輸出檔案名稱。[name] 會被替換為 entry 的鍵名(ex: background.js, manifest.js)
    path: path.resolve(__dirname, 'dist'), // 輸出目錄
  },
  mode: 'development', // 或 'production'
  module: {
    rules: [
      {
        test: /\.css$/,
        use: ['style-loader', 'css-loader'], // 如果需要處理 CSS 檔案
      },
      {
        test: /\.(png|svg|jpg|jpeg|gif)$/i,
        type: 'asset/resource', // 如果需要處理圖片檔案
      },
    ],
  },
};
```
#
#
#
#
##### 在 manifest.json 檔案中，將background.service_worker跟content_scripts.js替換成 Webpack 輸出的 dist/background.js & dist/manifest.js。
```json
  "background": {
    "service_worker": "./dist/background.js"
  }, 
  "content_scripts": [
    {
      "matches": [
        "<all_urls>"
	    ],
      "js": ["./dist/manifest.js"]
    }
  ],
```
---
### import. export測試
##### 安裝好Webpack後，我們馬上來測試看看import. export能不能使用!
#
#
#
#
##### 先建立一個untils.js檔案，裡面寫一個測試export funtion
```javascript
export function webpackTest()
{
    console.log("webpackTest");
}
```
{% asset_image webpack_test.jpg webpack_test %}
#
#
#
#
##### 接著在manifest.js裡面直接使用這個function。記得要先import untils.js這個檔案
```javascript
import { webpackTest } from './untils.js';

window.onload = function (){
    webpackTest();
}
```
#
#
#
#
##### 執行後應該是什麼事都不會發生，因為我們已經把程式的進入點改成./dist/manifest了，不是原本的./manifest。我們必須要先執行打包語法。
```bash
npx webpack
```
#
#
#
#
##### 打包完成後，可以看到dist裡面多了兩個檔案。
{% asset_image dist.jpg dist %}
#
#
#
#
##### 更新後，再重新執行一次擴充套件。可以看到import的webpackTest順利執行了!
{% asset_image test_result.jpg test_result %}