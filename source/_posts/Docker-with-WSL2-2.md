---
title: Docker with WSL2(2) - Dockerfile
date: 2025-09-25 16:26:33
categories:
  - Docker
tags:
  - WSL2
  - Ubuntu
  - DockerFile
---
##### 上篇文章我們在WSL裡安裝了Ubuntu. Docker Engin。現在就要透過Dockerfile來實作`[容器化]`
<!-- more -->
#
#
#
#
---
### 什麼是Dockerfile?
##### Dockerfile定義了如何建構 image 的步驟
``` dockerfile
FROM node:18
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
CMD ["npm", "start"]
```
#
#
##### 執行 docker build（或由 docker-compose 幫你 build 時）才會逐步執行這些指令 → 產生一個 image。

---
### 把原始檔案傳到WSL(Ubuntu Server)
##### 要進行容器化之前，必須要把在本機的專案資料夾內容裡的檔案，放到Ubuntu Server上
  * ##### 把原始檔案傳到Ubuntu Server有多種作法
    1. ##### 直接掛載 Windows 資料夾，WSL2 可以直接存取 Windows 資料夾：
    ``` bash
    /mnt/<磁碟機代號>/path/to/project
    ```
    ##### Windows 本機專案路徑：
    ``` text
    D:\Homer\Project_MyGit\my-form-app\
    ```
    {% asset_image ng_file_list.jpg ng_file_list%}
    ##### WSL2 對應路徑：
    ``` bash
        # 切換到專案目錄
        cd /mnt/d/Homer/Project_MyGit/my-form-app
    ```
    #
    #
    #
    #
    ``` bash
        # 查看專案檔案
        ls
    ```
    ##### 掛載成功，可以看到wsl成功取得my-form-app資料夾底下的檔案了!
    {% asset_image wsl2_mnt.jpg wsl2_mnt %}
    #
    #
    #
    #
    <span style="color:red">缺點：WSL2 對 /mnt 開頭的 Windows 資料夾 I/O 速度較慢。</span>
    #
    #
    #
    #
    2. ##### 複製專案到 WSL2 本地檔案系統：    
    ##### 創建 WSL2 專案資料夾（家目錄下）：
    ``` bash
        # 進入 WSL2 家目錄
        cd ~
        mkdir my-contact-form
    ```
    ##### 確認清空資料夾
    ``` bash
        cd ~/my-contact-form
        rm -rf ./*
    ```
    ##### 複製 Windows 專案到 WSL2
    ``` bash
        # 排除不需要的目錄
        rsync -av --progress \
        --exclude node_modules \
        --exclude .git \
        --exclude .vs \
        --exclude dist \
        /mnt/d/Homer/Project_MyGit/my-form-app/ ~/my-contact-form/
    ```
    ##### 確認檔案成功複製到Ubuntu的~/my-contact-form
    ``` bash
        cd ~/my-contact-form
        ls -l
    ```
    {% asset_image rsync_copy.jpg rsync_copy %}
    #
    #
    #
    #
    <span style="color:red">缺點：每次 Windows 更新專案後要同步（可以用 git 管理避免手動複製）。</span>        
    #
    #
    #
    #        
    3. ##### 使用 git 管理（最佳做法）
    #
    #
    #
    #        
    4. ##### 使用FTP傳輸也是可以的，但因為Ubuntu還外另外安裝FTP的服務，這邊就不另外作範例了。
#
#
#
#     
---
### 新增Dockerfile
##### Dockerfile 就是一個「指令腳本」，告訴 Docker 要怎麼一步一步去建構出一個映像檔 (image)。
##### Dockerfile = 一份建構「應用環境 + 程式」的腳本，讓你能快速、可重複地建立映像檔，再拿來跑容器。
#
#
#
#  
* ##### Dockerfile建立有下面兩種方式：
1. ##### 在本機的專案資料夾裡面手動建立. 編輯內容後，再複製到WSL專案資料夾裡。
2. ##### 直接在WSL專案資料夾裡用建立. 編輯內容。
#
#
#
# 
* ##### 這邊我們用第2.種方式來建立Dockerfile.
  1. ##### 先到專案目錄：
  ``` bash
  cd ~/my-contact-form
  ```
  2. ##### 新增（或編輯）Dockerfile：
  ``` bash
  nano Dockerfile
  ```
    ##### Dockerfile內容如下：
    ``` text
    # 1. Build 階段
    FROM node:18 AS build
    WORKDIR /app
    COPY package*.json ./
    RUN npm install
    COPY . .
    RUN npm run build --prod

    # 2. Runtime 階段
    FROM nginx:alpine
    WORKDIR /usr/share/nginx/html
    COPY --from=build /app/dist/my-contact-form .
    EXPOSE 80
    CMD ["nginx", "-g", "daemon off;"]    
    ```

    <h5 style="color: red;">這裡先解釋一下WORKDIR /app這個東西。WORKDIR 指的是容器裡的工作目錄，和 WSL 上的 ~/my-contact-form 沒有直接關聯，這裡的 /app 是容器裡的資料夾，跟 WSL 的 /home/weihao/my-contact-form 沒有關係!!</h5>

    * ##### Dockerfile Buid階段的逐行解說：
        * ##### FROM node:18 AS build
            * ##### 功能：指定此 stage 的基礎映像為 node:18，並把這個 stage 命名為 build（後面 --from=build 會用到）。
            * ##### 為什麼：需要 Node 環境來安裝 npm 套件並執行 ng build（編譯 Angular）。
            * ##### 建議：若想更小的映像可改用 node:18-alpine，但注意某些 native module 在 alpine 需額外步驟。
        * ##### WORKDIR /app
            * ##### 功能：設定容器內的工作目錄為 /app。
            * ##### 行為：之後的相對路徑（如 COPY . .、RUN npm install）都在 /app 下執行；若該目錄不存在會自動建立。
            * ##### 小提醒：<h5 style="color:red">WORKDIR 是容器內路徑，與你 WSL/Windows 的 ~/my-contact-form 無直接關係。</h5>
        * ##### COPY package*.json ./
            * ##### 功能：把主機的 package.json 與（若存在）package-lock.json 複製到容器的當前目錄（即 /app）。
            * ##### 為什麼先複製這兩個檔案：利用 Docker build cache。只有這兩個檔案變動時才會重新執行 npm install，能大幅加速重複 build。   
            
              <span style="color:red;font-weight:bold;">上面這段話聽起來很饒舌，我用我自己的方式來解釋一次：Dockerfile的每一行都會記錄成一個快取，只要文件或資料夾沒有異動。下次docker build時就會直接調用快取。所以先把package*.json複製到/app，只要package*.json沒有異動，就不會執行下一行的RUN npm install了，畢竟npm install會花很久的時間，這樣子能省下很多docker build的時間</span>
            
            * ##### 注意：package*.json 會同時匹配 package.json 與 package-lock.json（或 npm-shrinkwrap.json）。
        * ##### RUN npm install
            * ##### 功能：在容器中執行依賴安裝，產生 node_modules 供後續 build 使用。
            * ##### 建議：
              * ##### 若你有 package-lock.json，用 npm ci 更能確保可重現的安裝並且通常更快：RUN npm ci --only=production（但 npm ci 需要 lock 檔存在）。
              * ##### 把 node_modules 加入 .dockerignore，避免不必要的複製。
        * ##### COPY . .
            * ##### 功能：把整個專案（目前上下文目錄下的檔案）複製到容器 /app。
            * ##### 注意：
              * ##### 會複製 src/, angular.json, 等等；若有 .dockerignore（建議有），會排除 node_modules、dist、.git 等不需要的項目。
              * ##### 這行放在 npm install 之後能利用快取；若你把它放在前面，任何檔案變動都會重新跑 npm install，非常慢。   
        * ##### RUN npm run build --prod
            * ##### 功能：在容器內執行專案的生產編譯（Angular CLI），生成靜態檔到 dist/<project-name>（你例子是 dist/my-contact-form）。
            * ##### 注意：
              * ##### --prod 是構建 production 的捷徑（等同 --configuration production），不同 Angular CLI 版本行為等同，但若要保險可以寫 npm run build -- --configuration=production（取決於 package.json script 定義）。
              * ##### 編譯會產生 /dist/<outputPath>，後面 runtime stage 會用到。
              <h5 style="color:red;">這邊要注意的是npm run build是在容器裡的工作目錄(/app)執行的，所以在本機不會有build出來的dist資料夾!!!</h5>                                     

      * ##### Dockerfile Runtime階段的逐行解說：
        * ##### FROM nginx:alpine
            * ##### 功能：新的 stage，基礎映像為 nginx:alpine，只含 nginx，用來當靜態檔伺服器。
            * ##### 為什麼：multi-stage 的目的就是把 heavy build 環境（Node + node_modules）只保留在 build stage，最終 image 僅含靜態伺服器（小、快速、安全）。
        * ##### WORKDIR /usr/share/nginx/html
            * ##### 功能：設定 nginx 中靜態檔根目錄為工作目錄（nginx 的預設靜態路徑就是 /usr/share/nginx/html）。
            * ##### 因為我們下一行 COPY ... . 的目的地是當前目錄（.），所以要先設 WORKDIR 方便相對複製。
            <h5 style="color:red;">*WORKDIR /usr/share/nginx/html 這個資料夾是等docker build時，執行過FROM nginx:alpine才會有</h5>
        * ##### COPY --from=build /app/dist/my-contact-form .
            * ##### 功能：從 build stage（之前命名的 stage）拷貝 /app/dist/my-contact-form 的內容到目前的工作目錄（也就是 /usr/share/nginx/html）。   
            * ##### 注意：
              * ##### my-contact-form 必須和你在 angular.json 設定的 outputPath 相同；否則 COPY 路徑會找不到檔案而失敗。 --configuration=production（取決於 package.json script 定義）。
              * ##### . 代表目前 WORKDIR（/usr/share/nginx/html），所以複製完你 nginx 根目錄會直接有 index.html、main.js 等檔案。
        * ##### EXPOSE 80
            * ##### 標示此容器會使用的網路埠（文件性質）。
            * ##### 注意：
              * ##### EXPOSE 不會真正開放埠，只是做為鏡像的 metadata；實際要能外部連線要在 docker run -p 主機埠:容器埠 時對映。
        * ##### CMD ["nginx", "-g", "daemon off;"]
            * ##### 功能：容器啟動時執行的預設命令，這裡啟動 nginx 並以 foreground 模式運行（daemon off;）使 container 保持運行。
            * ##### 小提醒：用 JSON array 形式可避免 shell 解析問題。 
#
#
#
#     
---
### 建置 (Build) 映像檔

<h5 style="color: red;">如果有修改Angular的程式，記得一定要重新複製新程式到WSL!! 不然怎麼build都會是舊的!!</h5>

* ##### 編輯好Dockerfile後，接著要建置映像檔(image)
    * ##### 在專案目錄執行：
    ``` bash
    cd ~/my-contact-form
    docker build -t my-contact-form .
    ```
    ##### -t my-contact-form → 幫映像檔取名字 my-contact-form。
    ##### . → 代表使用當前資料夾的 Dockerfile 與檔案。

    * <h5>這時候遇到了一個以後要記得避免的雷，要記得把node_module加到.dockerignore裡面，不然就會跟我一樣，在RUN npm install這裡花很久的等待時間!!(來總共花了472秒才跑完)</h5>
    {% asset_image npm_install.jpg npm_install %}

    * ##### 出現了錯誤訊息。
      {% asset_image docker_build_error_1.jpg docker_build_error_1 %}
        * ##### 原來是我們的angular.json裡面的輸出路徑為`"outputPath": "dist/my-form-app",`。
        {% asset_image outputpath.jpg outputpath %}
        
        * ##### 這導致Dockerfile的這一行`COPY --from=build /app/dist/my-contact-form .`因為找不到要COPY的來源路徑而出錯。
          ##### 所以我們要把Angular.json的輸出路徑改成跟Dockerfile的一樣
          ##### 修改前：`"outputPath": "dist/my-form-app"`
          ##### 修改後：`"outputPath": "dist/my-contact-form"`

        * ##### 修改好後，再重新執行docker build，終於成功了!!
        ``` bash
        cd ~/my-contact-form
        docker build -t my-contact-form .
        ```
        {% asset_image docker_build_success.jpg docker_build_success %}

        * ##### docker build完成後，可以用下面指令，確認image。
        ``` bash
        docker images
        ```
        {% asset_image docker_image.jpg docker_image %}
#
#
#
#     
---
### 啟動 (Run) 容器
* ##### 在專案目錄執行：
  ``` bash
  docker run -d -p 8080:80 --name my-contact-form my-contact-form
  ```
    * ##### 參數用途：
      * ##### -d → 背景執行（detached mode）。
      * ##### -p 8080:80 → 把主機的 8080 port 對映到容器內的 80 port（nginx 預設 port）。
      * ##### --name my-contact-form → 幫容器取名字，方便管理。
      * ##### my-contact-form → 使用剛剛 build 出來的 image。
  #
  #
  #
  #
* ##### 連線測試：
  ##### 連線成功!
  {% asset_image docker_run.jpg docker_run %}
  #
  #
  #
  #
  ##### 來觀察一下container的狀態，狀態為UP。
  ``` base
  docer ps -a
  ```
  {% asset_image docker_ps.jpg docker_ps%}
#
#
#
#     
---
* ##### 這個系列到目前為止，已經完成了  
  1. ##### [在WSL上安裝了Ubuntu + Docker Engin，並且建立好快照備份了](https://19kiko88.github.io/2025/09/22/docker-with-WSL2-1/)  
  2. ##### 建立Dockerfile，以及Docker的建置以及啟動執行




      









