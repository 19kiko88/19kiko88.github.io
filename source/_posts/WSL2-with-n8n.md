---
title: WSL2 with n8n
date: 2026-06-24 13:38:47
categories:
  - AI應用
tags:
  - AI
  - n8n
  - docker
---
##### 紀錄透過WSL來啟用n8n docker container。
<!-- more -->
#
#
#
---
### 1.建立 n8n 工作目錄
``` bash
# 建立一個名為 n8n_local 的資料夾
mkdir -p ~/n8n_local

# 進入該資料夾
cd ~/n8n_local
```
#
#
#
---
### 2.使用 Docker Compose 啟動 n8n
* ##### 建立設定檔
``` bash
nano docker-compose.yml
```
#
#
#
* ##### 貼上以下設定，並按Ctrl+X儲存
``` yaml
version: '3.8'

services:
  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    container_name: n8n_local
    restart: always
    ports:
      - "5678:5678"
    environment:
      - TZ=Asia/Taipei # 設定你的時區
      - N8N_SECURE_COOKIE=false # 地端測試關閉安全 Cookie 限制
    volumes:
      - ~/n8n_local/n8n_data:/home/node/.n8n
```
#
#
#
* ##### 一鍵啟動容器
``` bash
docker compose up -d
```
#
#
#
---
### 3.開啟 n8n 網頁介面
##### 當看到終端機顯示 Started 之後，就可以打開瀏覽器，直接輸入以下網址：
``` text
http://localhost:5678
```
#
#
#
---
### 4.Trouble Shooting
##### 以上都是一切順利的情況，像我就無法正常開啟n8n，先看一下n8n的狀態，可以看到狀態為Restaring。問了一下AI，說是有遇到錯誤，一直嘗試重啟。
``` bash
docker ps
```
{% asset_image n8n_status.jpg n8n_status %}
#
#
#
##### 再看一下n8n的log
``` bash
docker compose logs n8n
```
#
#
#
##### 發現錯誤：n8n_local  | Error: EACCES: permission denied, open '/home/node/.n8n/config'。看來是Linux 檔案權限的問題。
  * ##### 先確認一下n8n_data資料夾的權限，顯示：drwxr-xr-x 2 root root 。表示目前只有root有權限存取。
    ``` bash
# 先切換到你的 n8n 工作目錄
cd ~/n8n_local

# 查看 n8n_data 資料夾的詳細權限
ls -ld n8n_data
    ```

  * ##### 修正 WSL2 資料夾的權限
    ``` bash
# 確保你目前在 ~/n8n_local 目錄下
cd ~/n8n_local

# 建立資料夾（如果剛才沒被成功建立的話）
mkdir -p n8n_data

# 【核心指令】將資料夾擁有者直接改成 UID 1000
sudo chown -R 1000:1000 n8n_data
    ```

  * ##### 重新啟動n8n
    ``` bash
docker compose down && docker compose up -d
    ```

  * ##### 在確認一次n8n狀態，狀態為：Up，就可以透過網址：[http://localhost:5678] 進入n8n了~
    ``` bash
docker ps
    ```
    {% asset_image n8n_status_up.jpg n8n_status_up %}

