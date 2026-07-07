---
title: n8n workflow包裝成MCP Service
date: 2026-07-03 11:23:05
categories:
  - AI應用
tags:
  - AI
  - n8n
  - docker
  - Side Project
--- 
##### 紀錄一下魔鏡專案的統整。
<!-- more -->
#
#
#
---
### WSL + n8n 環境架設
* ##### 主要處理下面這兩件事：
  1. ##### 在WSL裡啟用n8n container
  2. ##### docker-compose.yml設定
##### 詳細步驟可以參考：[WSL2 with n8n](https://19kiko88.github.io/2026/06/24/WSL2-with-n8x/)
#
#
#
---
### 拍照處理
* ##### 主要處理下面這兩件事：
  1. ##### 撰寫一個自動啟動相機前鏡頭，並拍攝照片的腳本(take_photo.ps1)。
  2. ##### 撰寫一個用node.js寫中介API程式(server.js)，讓n8n的webhook可以呼叫，並取得回傳值(照片的base64...etc)。

* ##### (Host)拍照功能測試
``` bash  
owershell.exe -ExecutionPolicy Bypass -File "D:\\Homer\\Project_MyGit\\mcp-camera\\take_photo.ps1" 
```
{% asset_image photo_success.jpg photo_success  %}
#
#
#
---
### 鏡像網路設定
##### 因為整個環境都是架設在公司的筆電上，因資安因素，導致n8n container一直無法呼叫到本機的API程式(server.js)來啟動拍照腳本。經查可能是因為port被阻擋的關係，在WSL預設的ufw原本就預設不開放的情形下，比較有可能是被Host的防火牆給阻擋port。
#
#
##### 最後是透過Mirror Network來解決這個問題，讓整個網路架構會被完全扁平化。WSL2不再擁有獨立的虛擬網卡(vEthernet)，而是直接共享、鏡像(Windows)Host的所有實體與虛擬網卡的網路環境。在整個網路環境都是一體的環境下，自然不會有什麼port被阻擋的問題。
#
#
##### 詳細步驟可以參考：[透過鏡像網路(Mirror Network)讓Host網路設定與Docker同步](https://19kiko88.github.io/2026/06/26/%E9%80%8F%E9%81%8E%E9%8F%A1%E5%83%8F%E7%B6%B2%E8%B7%AF(Mirror%20Network)%E8%AE%93Host%E7%B6%B2%E8%B7%AF%E8%A8%AD%E5%AE%9A%E8%88%87Docker%E5%90%8C%E6%AD%A5/)
#
#
#
---
### 設計 n8n WorkFlow
##### 簡單說明一下n8n的workflow流程：
##### 一開始是透過webhook來啟動，呼叫拍照api(server.js)取得照片資訊後，透過Google Cloud Vision分析出結果，在給LLM產出讚美的文字，再透過Google Cloud TTS把文字轉成語音，最後再把整個結果轉成網頁。並回傳URL給Client。
{% asset_image workflow.jpg workflow  %}
#
#
* ##### WorkFlow流程測試
  1. ##### 點選 Execute workflow  
  {% asset_image workflow_test.jpg workflow_test  %}
  2. ##### 因為是webhook啟動，必須在瀏覽器透過網址來啟動workflow(URL可以點選webhook進入設定裡面取得)
  3. ##### 取得回應結果  
  {% asset_image workflow_test_result.jpg workflow_test_result  %}
  4. ##### 設定docker-compose.yml，掛載html檔案的輸出目錄。
  ``` yaml
  - /mnt/d/Homer/Project_MyGit/mcp-camera/www:/www   # ✅ 新增：魔鏡 HTML 輸出目錄
  ```
  5. ##### 啟動一個Nginx靜態網頁伺服器Container，把 www 資料夾裡的 HTML/CSS/JS 透過 Nginx 對外服務，讓瀏覽器可以透過 http://localhost:8080 存取魔鏡介面。
  ``` bash
  docker run -d -p 8080:80 -v /mnt/d/Homer/Project_MyGit/mcp-camera/www:/usr/share/nginx/html nginx
  ```
  {% asset_image nginx_run.jpg nginx_run  %}  
  #
  #
##### 結果：
{% asset_image final_result.jpg final_result  %}
#
#
##### 補充，用GitHub幫WorkFlow做版控：[n8n with Git](https://19kiko88.github.io/2026/06/30/n8n-with-Git/)
#
#
#
---
### MCP Server 設定
* ##### 我們要把上一個流程的WorkFlow包裝成MCP，要先做一些相關設定。
  1. ##### 開啟n8n的MCP功能
  {% asset_image enable_mcp.jpg enable_mcp  %}
  #
  #
  {% asset_image enable_mcp_2.jpg enable_mcp_2  %}
  2. ##### 開啟workflow的MCP設定(也可以在workflow的schema json裡面直接設定)
  {% asset_image enable_workflow_mcp.jpg enable_workflow_mcp  %}
  3. ##### workflow的active => true(UI為Publsh按鈕，Publish必須變成綠燈。也可以在workflow的schema json裡面直接設定)    
  {% asset_image published.jpg published  %}
  4. ##### docker-compose.yml設定
  {% asset_image docker-compose-setting1.jpg docker-compose-setting1  %}
    ##### docker-compose.yml修改完後，記得要重啟!!
``` bash
    docker compose down
    docker compose up -d
```
#
#
#
---
### MCP Client 設定
##### 我們這邊的Client是用Claude Desktop(Chat)來呼叫的，要先設定claude_desktop_config.json的mcpService屬性。<span style="color:red;">設定完成後記得Claude Desktop一定要重開才會生效，不然不管怎麼改都沒用。</span>
#
#
1. ##### 原本是這樣設定mcpServers，但claude desktop一直無法成功連線到MCP。
``` json
"mcpServers": {
  "smart-mirror": {
    "command": "node",
    "args": [
      "C:\\Users\\homer_chen\\AppData\\Roaming\\npm\\node_modules\\mcp-remote\\dist\\cli.js",
      "http://localhost:5678/mcp",
      "--header",
      "X-N8N-API-KEY:YOUR_API_KEY"
    ]
  }
}
```
#
#
2. ##### 改成n8n提供的MCP連線字串一樣不行。
{% asset_image conn_details.jpg conn_details  %}
``` json
  "mcpServers": {
    "n8n-mcp": {
      "type": "http",
      "url": "http://localhost:5678/mcp-server/http",
      "headers": {
        "Authorization": "Bearer <YOUR_ACCESS_TOKEN_HERE>"
      }
    }
  }
```
#
#
##### 問了AI，為什麼使用n8n內建的MCP Server會無法讓MCP Client連線。
{% asset_image mcp_conn_error1.jpg mcp_conn_error1  %}
#
#
3. ##### 後來改用stdio的方式，另外寫一隻node.js(smart-mirror-mcp.js)程式來呼叫，終於連線成功
``` json
  "mcpServers": {
    "smart-mirror": {
      "command": "node",
      "args": [
        "D:\\Homer\\Project_MyGit\\mcp-camera\\smart-mirror-mcp.js"
      ]
    }
  }  
```
{% asset_image mcp_conn_success.jpg mcp_conn_success  %}
#
#
##### MCP Client 測試結果：
{% asset_image mcp_final_success.jpg mcp_final_success  %}
#
#
##### 問了AI，為什麼使用stdio就可以了。
{% asset_image mcp_conn_error2.jpg mcp_conn_error2  %}
#
#
#
---
### 注意事項
* ##### <span style="color:red;">以下是整個專案在執行前，必須要先執行的兩個指令。</span>
  1. ##### (Host)確認拍照API有啟動
``` bash
node server.js
```
#
#
  2. ##### (WSL)啟動一個Nginx靜態網頁伺服器Container，沒啟動的話會無法瀏覽最終產出的html網頁。
``` bash
docker run -d -p 8080:80 -v /mnt/d/Homer/Project_MyGit/mcp-camera/www:/usr/share/nginx/html nginx
```
#
#
#
---

```
###
---
### Demo



1. active => true(UI為Publsh按鈕)
2. availableInMCP => true
3.Local_MCP_servers設定
{% asset_image Local_MCP_servers.jpg Local_MCP_servers  %}


依序檢查這三點
1. 確認 n8n 正在運行
打開瀏覽器確認 http://localhost:5678 能正常開啟 n8n 介面。
2. 確認 workflow 已經 Publish(Publish必須為綠燈)
回到 n8n 畫布，確認右上角是 Published 狀態，不是草稿。
3. 確認 n8n 有開啟 MCP 功能
在 n8n 後台確認這個路徑能回應：
http://localhost:5678/mcp
直接在瀏覽器開這個網址，如果出現 401 或任何回應代表 MCP endpoint 有開，如果完全沒回應代表 n8n 沒有啟用 MCP Server 功能。
