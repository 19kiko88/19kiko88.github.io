---
title: Selenium in Linux
date: 2025-12-23 10:31:08
tags:
---
##### 先前有個SideProject，會用網路爬蟲收集其他網站的資料。之前用的方式是透過HttpRequestMessage物件來發送Request，再取得整個網頁內容。但如果網頁有使用ajax時，就要使用Selenium。原本在本機(Windows 11)測試時一切正常，但一放到遠端Server(Ubuntu 22)後，就問題一堆
<!-- more -->
#
#
#
---
---
##### 首先要在Linux上安裝Chrome & 相關依賴套件，chromedriver則是你的selenium版本 <= 4.6時，才需要安裝。如果缺少了這3樣東西，會出現下面這個錯誤：`session not created: Chrome instance exited` 代表 Selenium 已經找到了 Chrome，但 Chrome 在啟動時崩潰了。

``` text
System.AggregateException: One or more errors occurred. (One or more errors occurred. (session not created: Chrome instance exited. Examine ChromeDriver verbose log to determine the cause. (SessionNotCreated)))
```
##### 解法：
* ##### 在Linux Server上面安裝 Chrome 本體。先在 Linux 下執行一次 `google-chrome --version` 看看能不能跑，如果出現 `command not found`，代表你必須先安裝 Chrome。
``` bash
#!/bin/bash

# 1. 切換到 tmp 目錄 (確保權限開放)
cd /tmp

# 2. 檢查是否已經有下載過的檔案，有的話先刪除以免衝突
rm -f google-chrome-stable_current_amd64.deb

# 3. 使用 wget 重新下載最新版的 Chrome
echo "正在從 Google 伺服器下載 Chrome..."
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb

    # 4. 檢查檔案是否存在
    if [ -f "google-chrome-stable_current_amd64.deb" ]; then
        echo "下載成功，開始安裝..."
        
        # 5. 使用 apt 安裝，這會自動處理相依性
        # 如果遇到權限問題，apt 會調用 root 權限，但檔案在 /tmp 所以不會報 Permission Denied
        sudo apt update
        sudo apt install ./google-chrome-stable_current_amd64.deb -y
        
        # 6. 驗證結果
        echo "安裝完成，檢查版本："
        google-chrome --version
    else
        echo "錯誤：檔案下載失敗，請檢查網路連線。"
    fi
  ```
  #
  #
  #
  * ##### 除了安裝Chrome之外，Linux Server上還要另外安裝相關的依賴： 
  ```bash
    sudo apt-get update && sudo apt-get install -y \
        libnss3 \
        libatk-bridge2.0-0 \
        libxcomposite1 \
        libxdamage1 \
        libxrandr2 \
        libgbm1 \
        libasound2 \
        libpangocairo-1.0-0 \
        libxshmfence1
  ```
#
#
#
---
---
##### 如果你安裝的Selenium.Driver版本 > 4.6（截至目前，最新版本為4.39.xx），可以跳過這個問題，Selenium 4.6+ 之後會自己下載selenium-manager。

##### 如果你安裝的Selenium.Driver版本 <= 4.6，遇到下面的問題，主要是在告訴你 .NET 7 編譯過程中，Selenium 4.6+ 引進的 Selenium Manager（負責自動下載瀏覽器驅動程式的元件）在 Linux 環境下的二進位檔沒有被正確包含或找到。

``` text
System.AggregateException: One or more errors occurred. (One or more errors occurred. (Selenium Manager binary 'selenium-manager' was not found in the following paths:\n - /var/www/html/{專案名稱}/selenium-manager\n - /var/www/html/{專案名稱}/runtimes/linux/native/selenium-manager\n - /usr/lib/dotnet/shared/Microsoft.NETCore.App/7.0.19/selenium-manager))
```
##### 解法：
* ##### 在 Linux 上「直接安裝 chromedriver」
``` bash
sudo apt update
sudo apt install -y chromium-chromedriver
```
#
#
#
* ##### 確認安裝成功：
``` bash
chromedriver --version
```
#
#
#
---
---
##### .NET 7 / C# 的 ChromeOption 設定

* ##### 確認 Chrome 真的存在。
``` bash
which google-chrome
which google-chrome-stable
google-chrome --version
```

##### 只要其中一個有路徑，把那條路徑填進：
``` csharp
options.BinaryLocation = "這裡";
```

* ##### .NET 7 / C# ChromeOption 設定
``` csharp
using OpenQA.Selenium;
using OpenQA.Selenium.Chrome;

var options = new ChromeOptions();

// 必要設定
options.AddArgument("--headless=new");          // Linux 無 GUI 必開
options.AddArgument("--disable-dev-shm-usage"); // 避免共享記憶體問題
options.AddArgument("--no-sandbox");             // 就算非 root 也安全
options.AddArgument("--window-size=1920,1080");

// 強烈建議：明確指定 Chrome 路徑
options.BinaryLocation = "/usr/bin/google-chrome-stable";

using var driver = new ChromeDriver(options);

driver.Navigate().GoToUrl("https://www.google.com");
Console.WriteLine(driver.Title);
```
---
---
##### 如果上面該安裝的套件都安裝後，還是一直出現 session not created 的 issue 的話。建議直接看ChromeDriver的log比較快，Selenium回應的訊息很籠統，不同的問題有可能都是相同的錯誤回應。透過log找到問題點後，再用ChatGPT幫你Debug，不然debug會de到瘋掉!! 

* ##### 開啟 Selenium Manager + ChromeDriver log 來觀察錯誤。
``` csharp
var service = ChromeDriverService.CreateDefaultService();
service.EnableVerboseLogging = true;
service.LogPath = "chromedriver.log";

using var driver = new ChromeDriver(service, options);
```

* ##### 執行後查看：
``` bash
cat chromedriver.log
```
``` bash
nano chromedriver.log
```
#
#
#
---
---
##### publish code
``` bash
dotnet publish -c Release -r linux-x64 --self-contained true -p:PublishSingleFile=false
```


