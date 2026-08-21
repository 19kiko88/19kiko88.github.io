---
title: Angular 專案中的 `ng`、`npm`、`npx` 差異與使用方式
date: 2026-08-20 14:42:14
categories:
 - 前端開發
tags:
 - Angular
---
<!-- # Angular 專案中的 `ng`、`npm`、`npx` 差異與使用方式 -->
<!-- more -->

在 Angular 專案中，經常會看到以下指令：

```bash
ng build
npm run build
npx ng build
```

三者看起來很相似，但實際上負責的事情不同。理解它們的關係，可以避免在執行 Angular CLI 指令時產生混淆。

---

## 一、先理解三者的關係

可以先用下面的方式理解：

```text
Node.js
  └─ npm
      ├─ npm install
      ├─ npm run
      ├─ npm publish
      └─ npx
```

其中：

* `ng`：Angular CLI
* `npm`：Node Package Manager，負責套件管理及執行 `package.json` 的 scripts
* `npx`：用來直接執行 npm 套件所提供的 CLI 工具

簡單來說：

```text
ng
↓
Angular CLI

npm
↓
套件管理 + package.json scripts

npx
↓
CLI 執行工具
```

---

# 二、`ng` 是什麼？

`ng` 是 **Angular CLI（Command Line Interface）** 提供的命令。

例如：

```bash
ng build
ng test
ng serve
ng generate component user
```

這些都是 Angular CLI 的功能。

例如：

```bash
ng build
```

會執行 Angular 的建置流程：

```text
Angular TypeScript
        ↓
Angular Compiler
        ↓
Bundle
        ↓
Minify / Optimize
        ↓
dist/
```

最後通常會產生：

```text
dist/
└── your-project/
    └── browser/
        ├── main-xxxxx.js
        ├── chunk-xxxxx.js
        └── styles-xxxxx.css
```

因此：

> `ng build` 本質上就是「使用 Angular CLI 進行 Angular 專案建置」。

---

# 三、`npm` 是什麼？

`npm` 是 **Node Package Manager**。

它主要負責 Node.js 專案的套件管理，例如：

```bash
npm install
npm install lodash
npm uninstall lodash
npm update
```

另外，`npm` 也可以執行 `package.json` 裡定義的 scripts。

例如：

```json
{
  "scripts": {
    "start": "ng serve",
    "build": "ng build",
    "test": "ng test"
  }
}
```

因此可以執行：

```bash
npm run build
```

npm 會去找：

```json
"scripts": {
  "build": "ng build"
}
```

然後執行：

```bash
ng build
```

所以：

```bash
npm run build
```

實際上的流程是：

```text
npm run build
      ↓
讀取 package.json
      ↓
找到 scripts.build
      ↓
執行 "ng build"
      ↓
Angular CLI 建置專案
```

---

# 四、`npm run build` 與 `ng build` 的差異

假設 `package.json`：

```json
{
  "scripts": {
    "build": "ng build"
  }
}
```

那麼：

```bash
npm run build
```

和：

```bash
ng build
```

最後執行的 Angular build 可能是一樣的。

但兩者的概念不同：

### `ng build`

直接執行 Angular CLI：

```text
ng
↓
Angular CLI
↓
build
```

### `npm run build`

執行 `package.json` 裡的 script：

```text
npm
↓
package.json
↓
scripts.build
↓
ng build
```

因此，公司專案通常會要求：

```bash
npm run build
```

因為 `package.json` 可以定義完整的建置流程。

例如：

```json
{
  "scripts": {
    "build": "ng build --configuration production"
  }
}
```

這時：

```bash
npm run build
```

其實等同於：

```bash
ng build --configuration production
```

甚至 script 可以包含更多步驟：

```json
{
  "scripts": {
    "build": "ng build && npm run copy-assets"
  }
}
```

這時候直接使用：

```bash
ng build
```

就不一定等同於：

```bash
npm run build
```

因此在公司專案中，**如果專案文件要求使用 `npm run build`，最好不要自行改成 `ng build`。**

---

# 五、`npx` 是什麼？

`npx` 主要用途是：

> **直接執行 npm 套件提供的 CLI 工具。**

例如專案裡安裝了：

```json
{
  "devDependencies": {
    "@angular/cli": "^20.0.0"
  }
}
```

可以使用：

```bash
npx ng build
```

`npx` 會尋找目前專案中的 CLI：

```text
node_modules/.bin/ng
```

然後執行它。

概念上可以理解成：

```bash
npx ng build
```

≈

```bash
./node_modules/.bin/ng build
```

因此 `npx` 很適合用來執行「專案本身已經安裝的 CLI 工具」。

---

# 六、為什麼需要 `npx`？

假設你沒有全域安裝 Angular CLI：

```bash
npm install -g @angular/cli
```

你可能無法直接使用：

```bash
ng build
```

但是如果 Angular CLI 已經存在於專案的：

```text
node_modules/
```

就可以：

```bash
npx ng build
```

直接使用專案內的 Angular CLI。

這樣可以避免依賴電腦上安裝的全域版本。

例如：

```text
電腦全域 Angular CLI
        ↓
版本 20

專案 node_modules
        ↓
Angular CLI 20.3.x
```

使用：

```bash
npx ng build
```

通常可以讓你使用專案安裝的 CLI 版本。

---

# 七、`npx build` 是什麼？

這是很容易搞錯的地方。

```bash
npx build
```

**並不等於：**

```bash
ng build
```

也不等於：

```bash
npm run build
```

`npx build` 的意思比較接近：

> 「請幫我尋找並執行一個叫做 `build` 的 CLI。」

它和 Angular 沒有直接關係。

所以在 Angular 專案中，如果目的是執行 Angular build，通常不要寫：

```bash
npx build
```

而應該根據需求使用：

```bash
ng build
```

或：

```bash
npm run build
```

或：

```bash
npx ng build
```

---

# 八、三者放在一起比較

假設 `package.json`：

```json
{
  "scripts": {
    "build": "ng build",
    "test": "ng test",
    "start": "ng serve"
  }
}
```

那麼：

## `ng build`

```text
ng
↓
Angular CLI
↓
build
↓
建置 Angular 專案
```

---

## `npm run build`

```text
npm
↓
package.json
↓
scripts.build
↓
ng build
↓
建置 Angular 專案
```

---

## `npx ng build`

```text
npx
↓
尋找 ng CLI
↓
node_modules/.bin/ng
↓
Angular CLI
↓
build
↓
建置 Angular 專案
```

---

# 九、最簡單的記憶方式

可以記成：

| 指令              | 可以理解成                                |
| --------------- | ------------------------------------ |
| `ng build`      | 直接叫 Angular CLI 建置                   |
| `npm run build` | 執行 `package.json` 裡面的 build script   |
| `npx ng build`  | 使用 npx 執行 Angular CLI                |
| `npx build`     | 執行名為 `build` 的 CLI，不代表 Angular build |

---

# 十、Angular 專案常見用法

### 安裝專案依賴

```bash
npm install
```

### 啟動 Angular 開發伺服器

```bash
ng serve
```

或者：

```bash
npm start
```

前提是：

```json
{
  "scripts": {
    "start": "ng serve"
  }
}
```

### 建置專案

```bash
ng build
```

或者：

```bash
npm run build
```

### 執行單元測試

```bash
ng test
```

或者：

```bash
npm test
```

前提是：

```json
{
  "scripts": {
    "test": "ng test"
  }
}
```

### 使用專案內的 Angular CLI

```bash
npx ng build
```

---

# 十一、結論

最重要的是理解三者的角色：

```text
                Node.js 生態系
                      │
                     npm
               ┌──────┴──────┐
               │             │
          套件管理        npm scripts
               │             │
          npm install    npm run build
                             │
                             ↓
                          ng build
                             │
                             ↓
                        Angular CLI
```

而 `npx` 則是另一個與 npm 搭配使用的工具：

```text
npx
 ↓
尋找並執行 npm 套件提供的 CLI
 ↓
例如：
npx ng build
npx prettier
npx eslint
```

因此可以用一句話總結：

> **`ng` 是 Angular CLI；`npm` 是 Node.js 的套件管理工具，也能執行 `package.json` scripts；`npx` 則主要用來直接執行 npm 套件提供的 CLI。**

<span style="color:red;">文章內容透過ai整理產出</span>

