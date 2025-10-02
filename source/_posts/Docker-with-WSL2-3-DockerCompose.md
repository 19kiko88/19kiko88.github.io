---
title: Docker with WSL2(3) - Docker Compose
date: 2025-10-01 16:26:33
categories:
  - Docker
tags:
  - Docker
  - WSL2
  - Ubuntu
  - Docker Compose
---
##### 上篇文章我們在WSL裡建立Dockerfile，以及Docker的建置以及啟動執行。
##### 但如過按照上篇文章這樣的做法，因為我們是在Dockerfile裡面進行ng build的，所以我們每次只要一變動了Angular的程式，除了要再重新把程式rsync到wsl外，還要另外再執行一次docker build來更新image檔。
##### 因為重新 docker build 產生的新 image，和 正在跑的 container 是兩個不同的東西。所以目前使用的container也要停掉再用新的image來啟動一個新的Container。
##### 在一般的開發流程中，`[熱重載]`是個重要的功能，改好程式後，所見及所得就要能馬上看到結果。如果還要再做完上面這些繁雜的步驟才能看到程式修改後的結果，這樣子一點都不符合熱重載的精神!!
##### 為了達到熱重載，這篇就來說說Docker Compose。
##### 
<!-- more -->
#
#
#
#
---
### 建立Docker Compose檔案
1. ##### 執行下面指令，建立docker-compose.yaml
``` bash
nano docker-compose.yaml
```
#
  ##### volumes屬性我們就設成./，把WSL當前專案目錄的檔案都複製到容器的虛擬目錄/app_dockercompose下。這邊要特別注意 angular-dev的command，我們也要跟之前一樣做port的mapping(0.0.0.0 mapping到port4200)，並且因為我們沒有再容器裡面安裝Angular CLI，所以不能直接用ng serve，改使用npx ng(npx 會自動執行 node_modules 裡的 ng，避免全域安裝版本問題)。

``` yaml
services:
  angular-dev:
    image: node:18
    container_name: angular-dev
    working_dir: /app_dockercompose
    volumes:
      - ./:/app_dockercompose
    ports:
      - "4200:4200"
    command: sh -c "npm install && npx ng serve --host 0.0.0.0 --port 4200"

  angular-prod:
    image: nginx:alpine
    container_name: angular-prod
    volumes:
      - ./my-contact-form/dist/my-contact-form:/usr/share/nginx/html
    ports:
      - "8080:80"    
```

2. ##### 執行下面指令，啟動Docker Compose。
``` bash
# 啟動（開發 + 部署）
docker compose up

# 只跑開發環境
docker compose up angular-dev

# 只跑生產測試環境
docker compose up angular-prod
```
---
### Docker Compose測試
1. ##### Docker Compose啟動後，我們來做個測試看看。先隨邊變更一下Angular的UI內容
    {% asset_image angular_modify.jpg angular_modify%}

2. ##### 接著把更新過後的程式從本機複製到WSL裡的專案資料夾
``` bash
    # 排除不需要的目錄
    rsync -av --progress \
    --exclude node_modules \
    --exclude .git \
    --exclude .vs \
    --exclude dist \
    /mnt/d/Homer/Project_MyGit/my-form-app/ ~/my-contact-form/
```

3. ##### 重新執行compose up
    ``` bash
    # 啟動（開發 + 部署）
    docker compose up
    ```
4. ##### UI成功更新了!!
{% asset_image docker_compose_success.jpg docker_compose_success%}
5. ##### 觀察一下啟動中的container清單。新啟動了一個名稱為angular-dev的container
    ``` bash
    docker ps -a
    ```
    {% asset_image docker_ps.jpg docker_ps %}
#
#
#
#
---
### 熱重載
1. ##### 完成上面Docker Compose啟動的步驟後，接著來試試看能不能達成所謂的熱重載。我們先在Angular的原始碼做點小變更。
    {% asset_image hot_reload_test_1.jpg hot_reload_test_1 %}
    {% asset_image hot_reload_test_2.jpg hot_reload_test_2 %}

2. ##### 接著執行rsync把本機的程式複製到wsl的專案資料夾裡面
    ``` bash
        # 排除不需要的目錄
        rsync -av --progress \
        --exclude node_modules \
        --exclude .git \
        --exclude .vs \
        --exclude dist \
        /mnt/d/Homer/Project_MyGit/my-form-app/ ~/my-contact-form/
    ```
3. ##### 接著F5重整畫面，看看在有沒有被加上時間。結果卻是嗯...無法連線到此頁面。
    {% asset_image not_found.jpg not_found %}
4. ##### 透過下面指令，透過bash連線進去container(angular-dev)看看，結果卻是顯示`[container is not running]`
    ``` bash
    docker exec -it angular-dev bash
    ```
    {% asset_image docker_exec_it.jpg docker_exec_it%}
5. ##### 觀察一下啟動中的container清單。container的狀態為Exited !!
    ``` bash
    docker ps -a
    ```
    {% asset_image docker_ps_2.jpg docker_ps_2 %}   
6. ##### 後來問了ChatGPT才知道， 如果你是用下面的指令(前景模式)啟動Docker Compose。那麼你在關掉terminal視窗或是按ctrl + c 時，container就會停止，狀態變為Exited。
    ``` bash
    docker compose up
    ```
    {% asset_image trouble_shooting.jpg trouble_shooting%}
    ##### 再執行一次docker ps -a查看Container的狀態。狀態確實是變成Exited了
    ``` bash
    docker ps -a
    ```
    {% asset_image docker_ps_3.jpg docker_ps_3 %}
7. ##### 為了避免上述情況，我們必須改用(背景模式)啟動Docker Compose。在docker compose up後面加上-d。這樣Container就會在背景一直跑了。詳細測試流程可以看下面的流程圖。
    ``` bash
    docker compose up -d angular-dev
    ```
    {% asset_image docker_compose_success_with_parameter_d.jpg docker_compose_success_with_parameter_d%}
7. ##### Docker Compose改採背景模式啟動後，我們在按下F5重整網頁看看。可以看到UI已經更新了，原本的字串Version DockerCompose後面被加上了現在時間了。
    {% asset_image hot_reload_test_3.jpg hot_reload_test_3 %}
8. ##### 接著我們在變更UI，新增一個表單欄位，然後執行rsync更新程式。結果是左邊的UI頁面也熱重載及時更新了!!
    {% video 'hot_reload_test_4.mp4' %}



