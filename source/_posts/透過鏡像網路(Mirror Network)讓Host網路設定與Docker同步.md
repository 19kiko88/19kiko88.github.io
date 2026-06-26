---
title: 透過鏡像網路(Mirror Network)讓Host網路設定與Docker同步!
date: 2026-06-26 14:28:28
categories:
  - AI應用
tags:
  - AI
  - n8n
  - docker
---
##### 最近在研究n8n + MCP，做了個測試MCP想讓AI幫我開啟筆電的前鏡頭並拍照。但n8n一直無法成功呼叫在Host的MCP Server，經查可能是port被擋住的問題。
<!-- more -->
#
#
#
---
##### n8n的AI Agent的Http Request Tool要呼叫MCP時，用的是網址是： http://172.22.128.1:5011/api/camera/ ，但一直無法連線。port被擋住的問題，在公司電腦無法任意開放指定port的情況下，經AI給的建議，知道了有鏡像網路(Mirror Network)這種東西可以解決我目前的問題。

##### 架構大概是這樣：host(campera mcp server) => WSL => Docker => n8n Container

##### 補充1：<span style="color:red">事後回想，我一開始沒想到有沒有可能是WSL那邊的問題，下次有機會可以試試先設定ufw開放port!</span>
##### 補充2：<span style="color:red">後來經過測試，WSL的ufw預設是不開啟的，所以很大機率應該還是Host的問題</span>

#
#
#
##### 172.22.128.1為Host vEthernet網卡的IP
{% asset_image ipconfig.jpg ipconfig %}

##### Http Request Tool的設定
{% asset_image http_request_tool.jpg http_request_tool %}

##### 回應失敗
{% asset_image time_out.jpg time_out %}

#
#
#
---
### 鏡像網路(Mirror Network)

##### 在 WSL 2（Windows Subsystem for Linux 2）中，鏡像網路模式（Mirrored Network Mode）是微軟推出的一項重大網路架構變革。簡單來說，它讓 WSL 2 的 Linux 系統直接「鏡像（複製）」了 Windows 主機的網路介面。開啟這個模式後，Windows 和 WSL 2 之間不再有網路邊界，兩者宛如處在同一個同溫層中。

##### 開啟鏡像網路前： 當你在 n8n Http Request Tool裡輸入MCP Server API網址 http://localhost:5011/api/camera 時，n8n 尋找的是「n8n 容器內部」的 localhost：5011，而不是 WSL 或 Windows Host 的 localhost：5011。
##### 開啟鏡像網路後：當你開啟 networkingMode=mirrored 後，網路架構會被完全扁平化。WSL 2 不再擁有獨立的虛擬網卡(vEthernet)，而是直接共享、鏡像 Windows 的所有實體與虛擬網卡。

* ##### 開啟鏡像網路(Mirror Network)
  * ##### 進入目錄 C:\Users\'{UserName}'\
  * ##### 新增設定檔檔案 .wslconfig，並貼入下面指令。
``` text
[wsl2]
networkingMode=mirrored
hostAddressLoopback=true
```

##### 啟用鏡像網路後，再次呼叫 MCP Server (http://localhost:5011/api/camera) 應該要可行，但結果卻依然連線失敗。原因是：雖然我們在 Windows Host 開啟了鏡像網路（Mirrored Mode），讓 Windows 與 WSL 的 localhost 互通，但 Docker 容器預設仍擁有獨立的網路隔離空間。<span style="color:red">由於 Docker 方面還沒進行對應的設定（意即尚未配置 network_mode: "host"），導致目前只有單方面開放，n8n 容器內部依然無法穿透到外部的 localhost。</span>
#
#
#
---
### 更新docker-compose.yml內容

##### (修改前)docker-compose.yml
``` yaml
version: '3.8'

services:
  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    container_name: n8n_local
    restart: always
    ports:
      - "5678:5678" # host port-5678 : docker port-5678。
    environment:
      - TZ=Asia/Taipei # 設定你的時區
      - N8N_SECURE_COOKIE=false # 地端測試關閉安全 Cookie 限制
    volumes:
      - ~/n8n_local/n8n_data:/home/node/.n8n
```

##### (修改後)docker-compose.yml
``` yaml
version: '3.8'

services:
  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    container_name: n8n_local
    restart: always
    network_mode: "host" # 讓容器直接使用 WSL/Windows 的 localhost    
    #ports: 
    #  - "5678:5678"
    environment:    
      - TZ=Asia/Taipei # 設定你的時區
      - N8N_SECURE_COOKIE=false # 地端測試關閉安全 Cookie 限制
      # n8n預設port為5678，設定network_mode: "host"後，host與docker的port為共用的，不須再特別設定，除非想換別的port`。
      # - N8N_PORT=5678
    volumes:
      - ~/n8n_local/n8n_data:/home/node/.n8n
```
#
#
#
---
### 測試

##### 先在WSL裡面用curl進行測試，成功取得回應。
``` bash
curl -X POST http://localhost:5011/api/camera -H "Content-Type: application/json" -d '{"action":"capture"}'
```
{% asset_image curl.jpg curl %}

#
#
#
##### 下prompt，請AI Model幫我開啟相機並拍照。
{% asset_image prompt.jpg prompt %}

#
#
#
##### 成功取得照片的base64，<span style="color:red">可以看到url雖然用的是localhost，但因為開啟了鏡像網路，所以網路環境與Host是共通的。</span>
{% asset_image picture_success.jpg picture_success %}
#
#
#
---
### 快速總結（打通 Windows 與 WSL Docker 網路）
1. **Host(Windows/WSL)**：於 Windows Host 建立 `.wslconfig`，開啟鏡像網路（Mirrored Mode）。
2. **Docker 容器**：於 `docker-compose.yml` 加上 `network_mode: "host"`，共享宿主機網路。
#
#
#
---
* ##### REF：
  1. #####  [WSL2 with n8n](https://19kiko88.github.io/2026/06/24/WSL2-with-n8x/)

