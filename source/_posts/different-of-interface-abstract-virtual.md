---
title: different of interface, abstract, virtual.
date: 2026-01-13 10:15:24
categories:
 - 物件導向
tags:
 - OOP
---
##### 最近再把舊專案的程式碼進行重構，interface/abstract/virtual&override這三個東西，之前雖然已經研究過了，但時間一久又忘記了。趁著這次進行程式重構，又加上AI的輔助，重新寫了一篇對於的interface/abstract/virtual&override的使用方法&情境研究，加上之前的經驗，感覺對於OOP又更有一些感覺了，對於這些東西不再是鴨子聽雷。
<!-- more -->
#
#
#
---
* ### abstract class(抽象類別)？
    1. ##### 在類別宣告中使用 abstract 修飾詞，<span style="color:red">來表示某一類別只是要作為其他類別的基底類別，不是自行具現化(不可以被newe)。</span>要繼承的子類別需要共用欄位/屬性/方法實作（如 Driver、Options 建立與釋放、共用工具方法），可使用abstract class，讓子類別繼承共用邏輯，只需實作差異化部分（如 RunAsync），<span style="color:red">可避免重複程式碼，維護性高</span>。

    2. ##### abstract class中可以包含抽象成員（沒有實作，且必須在子類別中被覆寫）和完全實作的成員（例如一般方法、屬性和建構函式）。
    3. ##### <span style="color:red">abstract class 裡的 abstract method，子類別「一定要」用 override 實作，否則直接編譯失敗。</span>
    ``` csharp
    abstract class A
    {
        public abstract void Do(); //抽象成員，沒有實作，必須在子類別中被覆寫
    }
    ```
    ``` csharp
    abstract class B: A
    {
        public override void Do()
        {
            /// 實作內容
        }
    }
    ```
#
#
#
---
* ### virtual & override
    1. ##### 當abstract class裡的method為virtual簽章時，繼承的子類別可以透過override簽章來覆寫父類別的virtual method內的實作。子類別也可以透過base來直接呼叫父類別的virtual method實作。
    ``` csharp
    abstract class A
    {
        public virtual void Do()
        {
            /// 實作內容
        }
    }
    ```
    ``` csharp
    abstract class B: A
    {
        public override void Do()
        {
            /// 實作內容
        }
    }
    ```
#
#
#
---
* ### Interface
    1. ##### 只定義合約（方法/屬性），不提供任何實作。
    2. ##### 讓不同類型的 crawler 可以有完全不同的實作（ex： 由Selenium 替換成 Playwright，<span style="color:red">不依賴特定class，達到解藕目的。</span>）。
    3. ##### 適合只需規範行為、無共用邏輯時。
#
#
#
---
### 程式範例(網路爬蟲)
* ##### Interface：定義「我是一個爬蟲」
    ##### 上層模組只依賴 interface
    ``` csharp
    public interface IWebCrawler
    {
        Task CrawlAsync();
    }
    ```
    * ##### 不管爬蟲套件是 Selenium / Playwright
    * ##### 不管實作細節
    * ##### DI / 排程 / Mock 都靠它
#
#
#
* ##### Abstract class：定義共用流程(邏輯)骨架
    ``` csharp
    public abstract class SeleniumCrawlerBase : IWebCrawler
    {
        protected IWebDriver Driver;

        //不允許 override（Template Method）
        public async Task CrawlAsync()
        {
            try
            {
                InitDriver();
                await BeforeCrawlAsync();
                await CrawlCoreAsync();     // 強制子類別實作
                await AfterCrawlAsync();
            }
            catch (Exception ex)
            {
                LogError(ex);
                throw;
            }
            finally
            {
                Driver?.Quit();
            }
        }

        // 必須 override（abstract）
        protected abstract Task CrawlCoreAsync();

        // 可選 override（virtual）
        protected virtual void InitDriver()
        {
            var options = new ChromeOptions();
            options.AddArgument("--headless");
            Driver = new ChromeDriver(options);
        }

        // 可選 override
        protected virtual Task BeforeCrawlAsync()
            => Task.CompletedTask;

        // 可選 override
        protected virtual Task AfterCrawlAsync()
            => Task.CompletedTask;

        protected void LogError(Exception ex)
        {
            Console.WriteLine(ex);
        }
    }
    ```
    * ##### 實作 interface
    * ##### 提供可 override 的擴充點
#
#
#    
* ##### 子類別實作（override 使用範例）
    ##### Site A：只實作「必須的 abstract」
    ``` csharp
    public class SiteACrawler : SeleniumCrawlerBase
    {
        protected override async Task CrawlCoreAsync()
        {
            Driver.Navigate().GoToUrl("https://site-a.com");

            var title = Driver.FindElement(By.Id("title")).Text;
            Console.WriteLine(title);

            await Task.CompletedTask;
        }
    }
    ```
    * ##### CrawlCoreAsync為abstract成員，必須強制override。
    #
    #
    #
    ##### Site B：override virtual（可選）
    ``` csharp
    public class SiteACrawler : SeleniumCrawlerBase
    {
        protected override async Task CrawlCoreAsync()
        {
            Driver.Navigate().GoToUrl("https://site-b.com");

            var wait = new WebDriverWait(Driver, TimeSpan.FromSeconds(10));
            wait.Until(d => d.FindElement(By.CssSelector(".load-more"))).Click();

            await Task.Delay(2000);
        }

        protected override Task BeforeCrawlAsync()
        {
            Console.WriteLine("Site B login...");
            return Task.CompletedTask;
        }        
    }
    ```
    #
    #
    #
    ##### Site C：override + 呼叫 base（進階）
    ``` csharp
    public class SiteCCrawler : SeleniumCrawlerBase
    {
        protected override void InitDriver()
        {
            base.InitDriver(); //保留父類邏輯

            var options = (ChromeOptions)((ChromeDriver)Driver).Options;
            options.AddArgument("--disable-blink-features=AutomationControlled");
        }

        protected override async Task CrawlCoreAsync()
        {
            Driver.Navigate().GoToUrl("https://site-c.com");
            await Task.CompletedTask;
        }

        protected override Task AfterCrawlAsync()
        {
            Console.WriteLine("Site C cleanup");
            return Task.CompletedTask;
        }
    }
    ```
    #
    #
    #
    ##### 上層使用（只看 interface）
    ``` csharp
    public class CrawlJob
    {
        private readonly IWebCrawler _crawler;

        public CrawlJob(IWebCrawler crawler)
        {
            _crawler = crawler;
        }

        public Task RunAsync()
        {
            return _crawler.CrawlAsync();//使用父類別的CrawlAsync()，其內容無法變更。
        }
    }
    ```
    * ##### <span style="color:red">完全不知道 Selenium，只依賴於interface，可把實作透過相依注入(DI)隨時替換成Playwright達成與Selenium解藕。</span>
#
#
#
---
* ### 三者分工總結
    * ##### interface => 對外合約 / 解耦
    * ##### abstract class => 共用邏輯流程
    * ##### override => 覆寫覆類別的行為
#
#
#
---
* ##### Ref：
    1. #####  [abstract (C# 參考)](https://learn.microsoft.com/zh-tw/dotnet/csharp/language-reference/keywords/abstract)
