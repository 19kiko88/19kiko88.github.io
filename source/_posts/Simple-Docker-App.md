---
title: Simple-Docker-App
date: 2025-10-22 15:36:11
categories:
  - Docker
tags:
  - WSL2
  - Ubuntu
---
##### 這篇做個簡單的docker實作範例，後端我們使用.Net WebAPI，並有個api可以查詢天氣預報，前端則使用Angular。把這個簡單的查詢小程式容器化。
<!-- more -->
#
#
#
#
---
##### 整體的專案架構大致會如下：
``` text
/simple-docker-app
├── /DockerBackend            # .NET Web API 專案資料夾
│   └── Dockerfile
├── /frontend           # Angular 專案資料夾
│   └── Dockerfile
├── /nginx              # Nginx 設定檔資料夾
│   └── nginx.conf
└── docker-compose.yml  # Docker Compose 主設定檔
```
#
#
#
#
---
* #####  後端.Net WebAPI：
    1. ##### 建立一個後端.Net WebAPI專案，這裡我用的是.Net 8，專案建立後，Controller裡面會有一個預設的測試API。我們就直接拿來使用，也不再另外更改程式了。
    ``` text
    [HttpGet]
    public IEnumerable<WeatherForecast> Get()
    {
        return Enumerable.Range(1, 5).Select(index => new WeatherForecast
        {
            Date = DateOnly.FromDateTime(DateTime.Now.AddDays(index)),
            TemperatureC = Random.Shared.Next(-20, 55),
            Summary = Summaries[Random.Shared.Next(Summaries.Length)]
        })
        .ToArray();
    }
    ```
    ##### 到目前為止，檔案結構會如下：
    {% asset_image project_structure_2.jpg project_structure_2 %}
    #
    #
    #
    2. ##### 把目前WebAPI資料夾內容掛載到WSL2。
        1. ##### 這邊我們用掛載目錄的方式讓WSL2取得後端程式碼，掛載目錄： WSL2會自動將Windows磁碟機掛載到 /mnt/ 下。您只需導航到您的專案目錄：
        ``` bash
        cd /mnt/d/Homer/Project_MyGit/simple-docker-app/DockerBackend
        ```

        2. ##### 掛載成功後，可以直接在WSL2(Linux)瀏覽Windows資料夾裡面的檔案。
        {% asset_image driver_letter_mount.jpg driver_letter_mount %}

        ##### 其他把Windows檔案搬到WSL2的方法，可以參考之前的文章。[Docker with WSL2(2) - Dockerfile](https://19kiko88.github.io/2025/09/25/Docker-with-WSL2-2/)
    #
    #
    #
    3. ##### 建立Docker file
        1. ##### 在WSL進入掛載目錄的路徑
        ``` bash
        cd /mnt/d/Homer/Project_MyGit/simple-docker-app/DockerBackend
        ```

        2. ##### 新建立一個Dockerfile檔案，特別提醒的是，Docker file不用有副檔名。
        ``` bash
        nano Dockerfile
        ```

        3. ##### 把下面內容貼到編輯器裡後儲存，完成Dockerfile檔案。
        ``` text
            # =================================================================
            # 階段 1: Build 階段 (使用 SDK 映像建置和發佈程式)
            # =================================================================
            FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
            WORKDIR /src/DockerBackend

            # 1. 複製專案檔 (*.csproj)
            # 將 DockerBackend.csproj 複製到 WORKDIR (/src/DockerBackend)
            COPY DockerBackend.csproj .

            # 2. 還原套件
            # 這樣做可以利用 Docker 快取
            RUN dotnet restore

            # 3. 複製所有原始碼 (包含 Program.cs、Controllers、Properties等)
            # 確保包含所有用於編譯的檔案
            COPY . .

            # 4. 發佈應用程式到 /app/publish
            # 從當前工作目錄 (/src/DockerBackend) 發佈
            RUN dotnet publish -c Release -o /app/publish /p:UseAppHost=false

            # =================================================================
            # 階段 2: Final 階段 (使用更輕量級的 Runtime 映像來運行程式)
            # =================================================================
            FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
            WORKDIR /app

            # 1. 將 Build 階段發佈的檔案複製過來
            COPY --from=build /app/publish .

            # 2. 定義容器啟動時執行的命令
            ENTRYPOINT ["dotnet", "DockerBackend.dll"]
        ```

        4. ##### 完成後可以看到，後端程式碼的資料夾裡多了一個Dockerfile檔案
        {% asset_image docker_filer_created_windows.jpg docker_filer_created_windows %}
        #
        {% asset_image docker_filer_created_wsl2.jpg docker_filer_created_wsl2 %}
        #
    #
    #
    #        
    4. ##### 利用Dockerfile建置Docker image
        1. ##### 在WSL進入掛載目錄的路徑，並執行下面指令，docker-backend-api則是建立的映像檔的名稱。
        ``` bash
            # 注意：建置上下文就是當前目錄 (.)
            docker build -t docker-backend-api .
        ```
        ##### image建置完成了
        {% asset_image docker_build.jpg docker_build %}

        2. ##### 透過docker images指令，可查詢到剛剛建立的image
        ``` bash
            docker images
        ```
        #
        {% asset_image docker_images.jpg docker_images %}
    #
    #
    #        
    5. ##### 運行容器
        1. ##### 執行下面指令，執行Image啟動Container。
        ``` bash
        docker run -d -p 8080:8080 --name my-backend-test -e ASPNETCORE_URLS="http://+:8080" docker-backend-api
        ```
        #
        2. ##### 透過docker ps指令，可查詢運行中的Container。
        ``` bash
            docker ps -a
        ```
        #
        ##### docker container運行成功了!
        {% asset_image docker_ps.jpg docker_ps%}
        #
        ##### api測試，成功取得api回應!!
        {% asset_image api_test.jpg api_test%}

* ##### 後端API的部分到這裡算是完成了。這裡我們來整理一下容器化之後的優點：  
    1. ##### 開發環境的「在我電腦上可以執行」問題：
        ##### 這種情況一定不陌生，每個人的.Net SDK/Runtime版本可能不同，這就可能導致版本較舊的開發人員無法正常執行新專案。上述的.Net可以替換成PostgreSQL. Redis. 等等...，容器化後，可保持環境的一致性。
    2. ##### CI/CD
        ##### bbbbbbbbbbbbbbbbbbbbbbbb
    3. ##### 快速擴充
        ##### 假設電商有促銷活動，當發現流量快無法負荷時，只要在有安裝Docker的伺服器上，直接用image啟動容器即可及時分擔流量。不必像傳統VM，光是啟動時間就比Docker多了許多(VM為分鐘級，Docker則為秒級)，還要再複製程式碼以及設定WebServer...等繁雜程序。
#
#
#
#    
---
* #####  前端Angular：
#
#
#
#
---
* #####  建立Docker Compose：
    1. ##### 建立Dockerfile    
    ``` yaml  
    version: '3.8'
    services:
    # 1. 後端服務: .NET Web API (Kestrel 直接暴露)
    webapi:
        build:
        context: ./backend
        dockerfile: Dockerfile
        # 將容器內部的 8080 埠映射到主機的 8080 埠
        ports:
        - "8080:8080" # <-- 這裡直接暴露給外部
        environment:
        - ASPNETCORE_URLS=http://+:8080
        restart: always

    # 2. 前端服務: Angular (直接使用 Nginx 伺服靜態檔案)
    frontend-app:
        build:
        context: ./frontend
        dockerfile: Dockerfile # 此 Dockerfile 最終仍使用 Nginx 容器來伺服靜態檔案
        # 將容器內部的 80 埠映射到主機的 80 埠
        ports:
        - "80:80" # <-- 這裡直接暴露給外部
        restart: always                    
    ```
#
#
#
#    
---    