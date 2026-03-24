---
title: 避免設定檔內的機密資訊(ex：DB連線字串)外流
date: 2026-03-19 11:32:07
categories:
  - 資安
tags:
---
##### 從寫程式以來，我的觀念都是把DB連線字串加密後放到Web.config或是appsetings.json裡面。之後偶然在網路上看到文章，都一致不建議在設定檔裡面放密碼，問了一下AI後才發現，原來還有這麼多自己沒想到的細節。
<!-- more -->
#
#
#
---
* ### .Net Framework設定檔轉化 (Web.config Transformation)
    1. ##### 如果是舊的 .NET Framework (ASP.NET) 專案，可以透過設定檔轉化來隔離開發環境跟正是環境的連線字串設定。
        * ##### Web.config 留開發用連線字串。
        * ##### Web.Release.config 寫入取代邏輯（如下）：
        ``` xml
        <connectionStrings>
        <add name="MyDB" 
            connectionString="真正的正式連線字串" 
            xdt:Transform="SetAttributes" xdt:Locator="Match(name)"/>
        </connectionStrings>
        ```
#
#
#
---
* ### .Net 配置覆蓋 (Configuration Overriding)。
    ##### 在.Net裡面也有類似.Net Framework設定檔轉化的功能。.NET 預設會按照特定的順序讀取多個配置來源，預設的讀取順序通常如下：
    * #####
        * ##### appsettings.json (最基礎、通用的設定)
        * ##### appsettings.{Environment}.json (如 Production 或 Development)
        * ##### User Secrets
#
#
#
---
* ### 避免機密資訊(ex：連線字串)在Internet上裸奔
    ##### 快速看完上面.Net跟.Net Framework設定檔的用法後。要記得push程式時。建議<span style="color:red">別把Web.Release.config(appsettings.Production.json)推到遠端的git，避免機密資訊(ex：連線字串)被公開。</span> 
    * ##### 我有問過AI：我把repository設成private不行嗎? 以下是AI給我的回應，順便列上給大家參考：
        * ##### 「凡走過必留下痕跡」 (Git History)：如果你今天不小心把密碼推上去，然後發現不對勁，立刻刪除並再推一次 Commit，密碼依然存在於 Git 的歷史紀錄中。任何有權限存取這個 Repo 人，只要翻閱 git log 或回溯舊版本，就能輕易看到那個密碼。
        * ##### 權限擴散 (Least Privilege Principle)：開發專案的人（工程師）通常不需要知道正式環境（Production）的資料庫密碼。如果密碼在 Git 裡，代表所有能看到 Code 的人（包含實習生外包廠商、測試人員）都變相擁有了正式環境的通行證。一旦其中一個人的 GitHub 帳號被盜，或是電腦遺失，你的正式環境就大門敞開了。
        * ##### CI/CD 與自動化工具的弱點：現代開發流程會串接很多第三方工具（如 Jenkins, SonarQube, Snyk 漏洞掃描）。這些工具會 Clone 你的程式碼進行分析。如果密碼在裡面，這些工具的 Log或資料庫裡也會留下密碼紀錄。你的安全邊界不再只是 GitHub，而是所有接觸過這份程式碼的平台。
        * ##### 服務供應商的風險 (Supply Chain Attack)：即便你的 Repo 是 Private，你依然是把密碼託管給了 GitHub (Microsoft)。歷史上曾發生過多次 GitHub 內部人員濫用權限或系統漏洞導致Private Repo 外洩的事件。在大公司架構中，我們會假設**「第三方平台不可完全信任」。
    #
    #
    #
    * ##### 接著我又另外問了AI：如果我把連線字串改成加密後的字串也不建議嗎? 以下是整理過後，AI給我的回應：
        * ##### 解密金鑰放哪？ 如果你把加密後的密碼放 Git，卻把解密金鑰也放 Git，那加密等於白做。
        * ##### 程式碼風險：如果將金鑰寫死在 C# 程式碼中，駭客只需透過「反編譯（Decompile）」就能在幾秒鐘內挖出金鑰並解密。
        * ##### 演算法會過時：現在安全的 AES 或 RSA 加密，幾年後可能被破解。將加密密碼留存在版本控制中，等於是留給未來的駭客一個「離線破解」的機會。            
    #
    #
    #
    * ##### 接著我又另外問了AI：如果我把加密金鑰用另一份檔案存下，並把這份檔案加到git ignore裡呢? 以下是整理過後，AI給我的回應：
        * ##### 災難復原：萬一那份被 Ignore 的金鑰檔丟了（例如你換電腦或伺服器硬碟壞了），你連自己都解不開密碼。
        * ##### 程式複雜化：你的 C# 程式碼必須多寫一段「讀取金鑰 -> 讀取加密密碼 -> 執行解密」的邏輯。
        * ##### 維護麻煩：你每次換密碼，都要先加密、貼到 JSON。再確保金鑰檔也有同步更新。
#
#
#
---        
* ### 範例1：把各環境的密碼，存放在各自的appsettings.{執行環境}.json裡面
##### 看完上面AI給的建議後，拿自己的專案來當個範例來實作一下。
##### appsettings結構如下：
  * ##### appsettings.json
    * ##### appsettings.Dev.json
    * ##### appsettings.Stg.json
##### appsettings.json裡面放的是設定檔的模板，連線字串因為是模板，我們放空白字串即可。
``` json
{
    "ConnectionStrings": {
        "SBHConnection": ""
    },
}
```
##### appsettings.Dev.json裡面的連線字串放本機開發環境的連線字串。
``` json
{
    "ConnectionStrings": {
        "SBHConnection": "本機開發環境DB連線字串"
    },
}
```
##### appsettings.Stg.json裡面的連線字串放測試環境的連線字串。
``` json
{
    "ConnectionStrings": {
        "SBHConnection": "測試環境DB連線字串"
    },
}
```
##### 然後記得把appsettings.Dev.json跟appsettings.Stg.json這兩個檔案加入.gitignore裡面，避免連線字串被push到git遠端。改成上面做法後，雖然不是最好的方案。但至少可以先處理掉先前提到的四個問題：1.「凡走過必留下痕跡」 (Git History) 2.權限擴散 (Least Privilege Principle) 3.CI/CD 與自動化工具的弱點 4.服務供應商的風險 (Supply Chain Attack)。但比較麻煩的事情是，<span style="color:red;">當連線字串的內容有異動時，要記得手動更新appsettings檔案到遠端server上。</span>
#
#
#
---
### 範例2：把機密資料放到Server的環境變數檔裡面
##### 這邊還有另外一種作法提供給大家，把DB連線字串直接寫在Linux上的服務設定檔裡。
  * ##### Linux 端的服務設定檔 (.service)
    ##### 在 Linux 上，我們透過 systemd 來管理 .NET 服務。你需要在 /etc/systemd/system/ 目錄下建立一個服務檔（例如 my-web-app.service）。
    ``` bash
    [Unit]
    Description=我的 .NET Web 應用程式
    After=network.target

    [Service]
    # 程式執行的目錄
    WorkingDirectory=/var/www/my-app
    # 啟動指令 (指向你的 dll)
    ExecStart=/usr/bin/dotnet /var/www/my-app/MyProject.dll
    Restart=always
    # 如果服務崩潰，10 秒後重啟
    RestartSec=10
    KillSignal=SIGINT
    SyslogIdentifier=dotnet-example

    # --- 關鍵部分：環境變數注入 ---
    # 注意：使用雙下底線「__」對應 JSON 的層級
    Environment=ASPNETCORE_ENVIRONMENT=Stg
    Environment=ConnectionStrings__DefaultConnection="Server=127.0.0.1;Database=RealDb;User Id=sa;Password=YourComplexPassword123!;"

    # 建議使用非 root 帳號執行以增加安全性
    User=www-data

    [Install]
    WantedBy=multi-user.target 
    ```
  * ##### .NET 端的 appsettings.Stg.json
    ##### appsettings.Stg.json裡面的連線字串可以整個區塊拿掉，或是給空白值。
  * ##### 啟用與驗證步驟
      * ##### 設定好 .service 檔後，請執行以下指令使其生效：
        1. ##### <span style="color: red;">重新載入設定：sudo systemctl daemon-reload</span>
        2. ##### 啟動服務：sudo systemctl start my-web-app.service
        3. ##### 設定開機自啟：sudo systemctl enable my-web-app.service
        4. ##### 檢查是否成功讀取變數：你可以透過 systemctl 查看該 Process 的環境變數（需 root 權限）：
        ``` bash
        # 找到 PID
        pid=$(systemctl show -p MainPID my-web-app.service | cut -d= -f2)
        # 查看該進程的環境變數
        sudo cat /proc/$pid/environ | tr '\0' '\n'
        ```

##### 改成這個方案後，又多了下面幾個優點：
  1. ##### 不留痕跡：伺服器上的 appsettings.json 就算被駭客偷走，裡面也只有假密碼。
  2. ##### 權限隔離：只有擁有 sudo 權限的人可以看 /etc/systemd/system/ 下的設定。
  3. ##### 防止誤推：開發者在本地開發時用自己的 User Secrets，推上 Git 時不用擔心漏刪密碼。
  ##### 整體而言，還是比上面的範例還要好。
