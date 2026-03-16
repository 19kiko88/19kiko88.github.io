---
title: Create Docker Volume
date: 2026-03-16 11:23:49
categories:
  - Docker
tags:
  - Docker
  - Docker Volume
---
##### 自己有個Side Project，資料庫是架在VPS上的Linux用Docker運行。當初為了求快，設定都用預設的(登入帳號用sa. 連線的port使用1433....etc)。最近看一下Log，發現被不明人士大量連線，嘗試破解密碼。問一下AI後，建議我不用要預設的sa帳號以及預設的1433 port。改帳號比較簡單，但要改Port的話就比較麻煩了。因為 Docker 啟動後無法直接修改 Port 映射，必須刪除舊容器並重新 run 一個新的。由於當初沒掛載 Volume，直接刪除容器會導致資料永久遺失。 為了保全資料，我必須先將資料備份出來，並在建立新容器時設定 Volume 掛載，確保未來即使再更換容器設定，資料也能持久保存。
<!-- more -->
#
#
#
---
* ### 備份資料庫
    ``` bash
    docker exec -it <舊容器名稱> /opt/mssql-tools18/bin/sqlcmd \
    -S localhost -U sa -P '你的舊密碼' \
    -C -Q "BACKUP DATABASE [你的資料庫名] TO DISK = N'/var/opt/mssql/data/backup.bak' WITH NOFORMAT, NOINIT, NAME = 'Full Backup', SKIP, NOREWIND, NOUNLOAD, STATS = 10"
    ```
    1. ##### Docker 外殼部分
        * ##### docker exec: 這是 Docker 的指令，用來進入一個「正在執行中」的容器執行後面的動作。
        * ##### -it:
            * ##### -i: (interactive) 讓你與容器互動。
            * ##### -t: (tty) 分配一個虛擬終端，讓你能在畫面上看到執行的過程（例如看到備份進度 %）。
        * ##### docker exec: 這是 Docker 的指令，用來進入一個「正在執行中」的容器執行後面的動作。
    2. ##### SQL 工具與登入部分
        * ##### /opt/mssql-tools18/bin/sqlcmd: 這是 SQL Server 在 Linux 版中內建的指令列管理工具（sqlcmd）的完整路徑。
        * ##### -S localhost: 指定 Server，因為我們已經在容器內部了，所以用 localhost。
        * ##### -U sa -P '你的舊密碼': 指定登入帳號（sa）與密碼。
    3. ##### SQL 指令核心 (-Q 後面的內容)
        ##### -Q 代表 "Query"，意思是「執行完括號內的 SQL 指令後就退出」。
        * ##### BACKUP DATABASE [你的資料庫名]: 指定要備份哪一個資料夾。
        * ##### TO DISK = N'/var/opt/mssql/data/backup.bak':
            * ##### 重點： 這是指容器內部的路徑。
            * ##### 它會把備份檔放在容器裡的 /var/opt/mssql/data/ 資料夾下，檔名為 backup.bak。
        * ##### WITH NOFORMAT, NOINIT...: 這些是備份的參數：
            * ##### NAME = 'Full Backup': 給這次備份取個名字。
            * ##### STATS = 10: 每完成 10% 就顯示一次進度（讓你知道它沒當機）。

        {% asset_image backup_db.jpg backup_db %}

    4. ### SQL Server 在 Linux 容器中預設存放資料與備份檔的地方
        ##### 如果你是為了確認備份要放哪裡，或者是想看資料庫檔案（.mdf, .ldf）在哪，可以執行下面指令(容器名稱可以透過docker ps -a查詢)：
        ``` bash
        docker exec -it <容器名稱> ls -lh /var/opt/mssql/data/
        ```

    5. ### 確認備份完成
        {% asset_image check_backup_file.jpg check_backup_file %}

    6. ### 將備份檔從容器複製到實體主機：
        ``` bash
        docker cp <舊容器名稱>:/var/opt/mssql/data/backup.bak ./backup.bak
        ```
        {% asset_image copy_backup_to_root.jpg copy_backup_to_root %}        
#
#
#
---
* ##### 新增VPS的對外Port
    1. ##### 先前提到，不使用預設的1433 Port來直接連線DB，所以我們要在VPS開一個新的對外Port來Mapping到1433。
        ``` bash
        netstat -tunlp | grep [要開放的VPS對外Port]
        ```

    2. ##### 使用 netstat (如果沒安裝，可能需要 apt install net-tools)
        ``` bash
        apt install net-tools
        ```

    3. ##### 開通後可用下面指令確認。
        ``` bash
        ufw status
        ```
#
#
#
---
* ### 建立 Volume
    1. ##### 手動建立 Volume。
        ``` bash
        docker volume create sbh_db_volume
        ```
        ##### <span style="color:red">sbh_db_volume</span>為Volume名稱，之後掛載時會用到。

    2. ##### 檢查是否建立成功：
        ``` bash
        docker volume ls
        ```

    3. ##### 啟動新容器（換 Port、掛載 Volume）：
        ``` bash
        docker run -e "ACCEPT_EULA=Y" -e "MSSQL_SA_PASSWORD=密碼" \
        -p [要開放的VPS對外Port]:1433 --name mssql_sbh \
        -v sbh_db_volume:/var/opt/mssql \
        -d mcr.microsoft.com/mssql/server:2022-latest
        ```
        * ##### --name：給這個新容器一個好記的名字，避免像之前系統隨機產生的。
        * ##### -d (detached): 讓容器在背景執行。如果沒有加這個，你關掉終端機（SSH）時容器就會跟著關掉。
        * ##### -v sbh_db_volume:/var/opt/mssql:
            * ##### sbh_db_volume: 之前先建立好的 Docker Volume（像是一個獨立的虛擬硬碟）。
            * ##### /var/opt/mssql: <span style="color:red">容器內</span>存放所有資料庫檔案的根目錄。
            * ##### mcr.microsoft.com/mssql/server:2022-latest: 指定從微軟官方倉庫下載 2022 最新版的 SQL Server 映像檔。

    4. ##### 確認容器是否啟用
        ```bash
        docker ps -a
        ```
        {% asset_image check_docker_run.jpg check_docker_run %}

    5. ##### 透過新Port連線至DB，確認DB連線正常。(因還未完成掛載流程，所以裡面還沒有資料)
        {% asset_image check_DB_run.jpg check_DB_run %}



##### <span style="color:red">之前資料是存在容器內部(可寫層)裡，Container刪除後，資料就沒了。</span>現在把資料存在外部實體空間裡面，並掛載至容器，以後無論你怎麼更新、刪除、重新啟動容器，只要掛載這個 Volume，資料（.mdf/.ldf）和登入帳號設定都會永遠留著。
#
#
#
---
* ### 將備份還原到新環境(容器)
    1. ##### 執行指令把備份檔複製到
        ``` bash
        docker cp ./backup.bak mssql_sbh:/var/opt/mssql/data/backup.bak
        ```
        * ##### docker cp (命令核心)：
            * ##### cp 代表 Copy。
            * ##### 這是 Docker 專門用來在 主機 (Host) 與 容器 (Container) 之間互相複製檔案的指令。它不需要容器開啟任何 Port，也不需要 SSH，只要 Docker 服務有跑就能動。
        * ##### ./backup.bak (來源路徑)
            * ##### VPS 主機上的檔案路徑。
            * ##### ./ 代表當前目錄。這就是你剛才從舊容器撈出來，或是剛上傳到 VPS 的那個資料庫備份檔。 
        * ##### mssql_new: (目標容器名稱)
            * ##### 指定你要把檔案丟進哪一個容器。
            * ##### 注意後面的 冒號 :，它是用來分隔「容器名稱」與「內部路徑」的必要符號。       
        * ##### /var/opt/mssql/data/backup.bak (目標存放路徑)
            * ##### <span style="color:red">這是容器內部的絕對路徑。</span>
            * ##### /var/opt/mssql/data/ 是 SQL Server 在 Linux 容器中預設最容易讀取檔案的地方。你把備份檔放在這裡，等一下進 SQL 執行還原指令（RESTORE）時，資料庫引擎才找得到它。
    {% asset_image copy_success.jpg copy_success %}

    2. ##### 執行資料庫還原語法
        ``` sql
        -- 1. 先確認備份檔內的邏輯檔案名稱 (LogicalName)
        RESTORE FILELISTONLY FROM DISK = '/var/opt/mssql/data/backup.bak';

        USE [master];
        RESTORE DATABASE [你的資料庫名] 
        FROM DISK = N'/var/opt/mssql/data/backup.bak' 
        WITH FILE = 1, NOUNLOAD, REPLACE, STATS = 5;
        ```
        * ##### Access Denied的問題：
            {% asset_image access_denied.jpg access_denied %}
            * ##### 以 root 身份進入 Container，預設情況下，docker exec 可能會沿用 mssql 的低權限帳號，所以你沒辦法 chown。請強制以 root 身份進入：
                ``` bash
                docker exec -u 0 -it <容器名稱或ID> /bin/bash
                ```
            * ##### 在容器內執行標準權限校正：
                ``` bash
                # 強制將擁有者改回 mssql (SQL Server 的執行帳號)
                chown mssql:root /var/opt/mssql/data/backup.bak
                ```
                ``` bash
                # 設定標準的讀寫權限 (擁有者讀寫，其他人唯讀)
                chmod 664 /var/opt/mssql/data/backup.bak
                ```
            * ##### 驗證權限狀態                          
                ``` bash
                ls -lh /var/opt/mssql/data/backup.bak
                ```              
        {% asset_image restore_success.jpg restore_success %}            
            



最終檢查清單
[ ] 刪除舊容器： 確認新容器資料都對了，再執行 docker rm -f <舊容器>。

[ ] 驗證 sa： 進入新容器後，立刻按照我們先前討論的，建立一個新管理員帳號並 停用 sa。

[ ] 防火牆： 記得在 VPS 的防火牆開啟 54321 並關閉 1433。        