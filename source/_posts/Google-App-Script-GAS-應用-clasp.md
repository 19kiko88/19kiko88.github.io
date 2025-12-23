---
title: Google App Script(GAS)應用 - Clasp
date: 2025-12-03 14:32:25
categories:
  - 自動化
tags:
  - GAS
  - Clasp
---
##### Clasp（Command Line Apps Script Projects）是 Google 官方推出的工具，用來讓你在本機撰寫 Apps Script。把 Apps Script（GAS）當成一般程式專案來寫、上傳、版本控制。
<!-- more -->
#
#
#
#
---
1. ##### 進入到 [App Script的管理頁](https://script.google.com/home?hl=zh-tw)
2. ##### 必須安裝 Node.js 6.0.0 以上版本，才能使用 Apps Script CLI (clasp)
3. ##### 執行指令，安裝clasp。
```bash
npm i @google/clasp -g
```
#
#
#
#
{% asset_image clasp_1.jpg  install clasp %}
#
#
#
#
4. ##### 安裝好clasp，先登入google帳號
    1. ##### 執行指令
    ```bash
    clasp login
    ```
    #
    #
    #
    #
    2. #####  會開啟一個登入頁面，要你登入google帳號。
    {% asset_image clasp_2.jpg  login google %}
    #
    #
    #
    #
    3. #####  設定全線開放，我目前是只先開啟最下面的3個項目。
    {% asset_image clasp_3.jpg  access_permission %}
    #
    #
    #
    #
    4. #####  登入成功。
    {% asset_image clasp_4.jpg  Logged_in %}
    #
    #
    #
    #
    5. #####  登入成功後，記得要到[App Script的管理頁設定](https://script.google.com/home/usersettings)，開啟Google Apps Script API。
    {% asset_image clasp_5.jpg  enable_Google Apps Script API %}
    #
    #
    #
    #
5. ##### 建立本地專案
    1. ##### 先移動到要建立專案的目錄
    ```bash
    d:
    ```
    ```bash
    cd  Project_MyGit\GoogleAppScript
    ```
    #
    #
    #
    #    
    2. #####  建立專案目錄
    ```bash
    mkdir clasp_condelab
    ```
    ```bash
    cd clasp_condelab
    ```
    ```bash
    clasp create --title "Clasp Codelab"  --type standalone
    ```
    ##### 如果沒有開啟4.5.的Google Apps Script API，會出現下面的錯誤。
    {% asset_image clasp_6.jpg  clasp create error %}
    #
    #
    #
    #
    3. #####  建立專案目錄完成後，會在目錄裡面多了兩個檔案(.clasp.json & appsscript.json)。
    {% asset_image clasp_7.jpg  clasp create success %}
    #####  到Google App Script的管理頁看，也會多了一個叫clasp_condelab的新專案。
    {% asset_image clasp_8.jpg  new project %}
    #
    #
    #
    #    
6. ##### push & pull & version 指令
    1. ##### 接著來試試看pull功能，我們在專案clasp_condelab的線上編輯器新增一個test檔案(不用副檔名)。並在裡面新增內容。編輯完成後要記得點選存檔!!
        {% asset_image clasp_9.jpg  add new file %}
        #
        #
        #
        #    
        {% asset_image clasp_10.jpg  add new file content %}
        #
        #
        #
        #    
        ##### 接著在本機執行pull指令，可以看到pull完成後，本機也多了一個在線上新增的test.gs檔案了。
        ```bash
        clasp pull
        ```
        {% asset_image clasp_11.jpg pull done. %}
        #
        #
        #
        #    
    2. ##### 如果執行push的話，則是會在把你在本機編輯的GAS程式push到遠端上面。我們編輯一下本機GAS程式，並執行push來做個測試。
        {% asset_image clasp_12.jpg %}
        #
        #
        #
        ```bash
        clasp push
        ```
        #
        #
        #
        ##### 重新整理GAS後，可以看到線上的GAS程式被改變了
        {% asset_image clasp_13.jpg %}
    3. ##### 如果執行version "{備註}"的話，則是會在線上GAS上新增一個快照版本，可以把目前的程式內容做成一份快照存下來。我們編輯一下線上GAS程式，並執行version來做個測試。
        {% asset_image clasp_14.jpg %}
        #
        #
        #    
        ```bash
        clasp version "test"
        ```
        {% asset_image clasp_15.jpg %}
        #
        #
        #
        ##### 切換到專案紀錄頁籤可以看到多了一份快照，幫你把目前的程式版本做了一份快照備份。
        {% asset_image clasp_16.jpg %}
    #
    #
    #
    #
    #
    ##### 最後，做個clasp push跟clasp version的比較
      * ##### clasp push = 把本地程式碼 → 推上 GAS（覆蓋最新草稿）
      * ##### clasp version "test" = 把目前線上程式碼 → 建立一個「命名版本（快照）」
    <span style="color:red">最後的最後，clasp pull 只能抓取 Apps Script 的最新版本（HEAD）程式碼，不能指定版本!! Gemini可能會跟你說可以這樣用：clasp pull -v {版本號碼}，不要被騙了!!</span>
    #
    #
    #
    #
    #
7. ##### 版本管理與部署，clasp 可讓您管理版本和部署作業。首先介紹一些詞彙：
    * ##### 版本：「快照」編寫程式碼版本可視為用於部署作業的唯讀分支版本。
    * ##### 部署作業：指令碼專案的已發布版本 (通常是外掛程式或網頁應用程式)。需要版本號碼。
    1. ##### 建立快照前，對本機程式做些修改並存檔。
    
    #
    #
    #
    #    
    2. ##### 接著執行指令，建立快照
    ```bash
    clasp version "version 2.0"
    ```
    ```bash
    clasp deploy 1 "First deployment"
    ```

#
#
#
#
---  

* ##### Ref：
    1. #####  [clasp - Apps Script CLI](https://codelabs.developers.google.com/codelabs/clasp?hl=zh-tw#0)
    2. #####  [Google Apps Script Local Development Tutorial](https://david-barreto.com/google-app-script-local-development-tutorial/)
