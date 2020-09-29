**前言**
最近有個需求是固定時間取得特定資料進行修改，在查詢相關資料之後決定使用 ASP.NET Core Generic Host 為出發，在搭配 .NET 中熱門的排程套件 Quartz.Net，測試完畢之後再將程式註冊為 Windows Service 服務就可滿足使用者的需求，這篇文章是整理開發時的重點流程為系列文，給有需要使用 ASP.NET Core 開發排程相關應用程式需求的朋友一些參考，若有問題或是錯誤的地方歡迎各位高手給予指導。

**Generic Host**
在過去處理 HTTP 請求時可以使用 WebHostBuilder 類別來處理，在開發 ASP.NET Core 應用程式設定及建立 WebHost 以及使用 Kestrel 服務接收請求並處理應用程式的生命週期，使用 DI 設定或配置 Logging 等重要功能，但並非所有的需求都依賴於 HTTP ，如 BackgroundService 背景執行程序或是處理 Queue 的資料等工作就不需要依賴於 HTTP 請求，從 ASP.NET Core 2.1 開始提供 **[Generic Host](https://docs.microsoft.com/zh-tw/aspnet/core/fundamentals/host/generic-host?view=aspnetcore-2.2) (泛型主機)**，提供開發者可以有新的選項來開發沒有 HTTP 請求的應用程式，可以透過 MSDN 示意圖了解

![](https://1.bp.blogspot.com/-lv-AbQZjL-8/XS5OtVYIVgI/AAAAAAAAHQM/nJbgQoLZ27AsdwrF3RsYR7Yz3EywpmCHACLcBGAs/s1600/IHostedService.png)

**CreateHostedBuilder**
如果選擇 Generic Host 開發程式，建議先了解 WebHostBuilder 背後預設配置的設定方式以及做了哪些事情，在之前 [ASP.NET Core 環境佈署設定 appsettings.json](https://marcus116.blogspot.com/2019/04/netcore-aspnet-core-appsettingsjson.html) 文章中有介紹過，建立新的 ASP.NTE Core 應用程式預設模板可以看到在 program.cs 中內建 CreateWebHostBuilder，此方法代碼中設定很多預設的配置，像是指定讀取資料夾中的 appsetting.json、環境變數、logging 設定、使用 Kestrel Server等預設內容。了解 WebHostBuilder 是如何設定的也有了基本概念，就可以發現有部分內容在 HostBuilder 是可以共用的，就可以開始著手開發 Generic Host 程式，首先先建立新的 ASP.NET Core Console 專案，並開啟 Program.cs 建立 CreateHostBuilder 運算式，代碼如下

```C#
private static IHostBuilder CreateHostBuilder() =>
    new HostBuilder()
        .ConfigureHostConfiguration(configHost =>
        {
            // setup Host configuration
            configHost.SetBasePath(Directory.GetCurrentDirectory())
                .AddJsonFile("hostsettings.json", optional: true)
                .AddEnvironmentVariables();
        })
        .ConfigureAppConfiguration((hostContext, configApp) =>
        {
            // setup App configuration
            configApp.AddJsonFile("appsettings.json")
                .AddJsonFile(
                    $"appsettings.{hostContext.HostingEnvironment.EnvironmentName}.json",
                    optional: true, reloadOnChange: true)
                .AddEnvironmentVariables();
        })
        .ConfigureServices((services =>
        {
            services.Configure<HostOptions>(option =>
            {
                option.ShutdownTimeout = TimeSpan.FromSeconds(30);
            });
        }))
        .ConfigureLogging((hostingContext, logging) => 
        {
            // Get Config from Logging section
            logging.AddConfiguration(hostingContext.Configuration.GetSection("Logging"))
                    .AddEventLog();
 
            if (hostingContext.HostingEnvironment.IsDevelopment())
            {
                logging.AddConsole();
            }
        }); 
```

將 TimedHostedService 加入到 Service 中

```C#
.ConfigureServices((hostContext, services) =>
{
    // ...
    services.AddHostedService<TimedHostedService>();
}
```

在 main 方法中直接使用 RunConsoleAync 

```C#
static async Task Main(string[] args)
{
    try
    {
        var builder = CreateHostBuilder();
 
        await builder.RunConsoleAsync();        
    }
    catch (Exception e)
    {
        Console.WriteLine(e);
        throw;
    }
}
```

這邊需要注意的是由於 async 是 C# 7.1 開始支援的功能，因此如果專案預設編譯需要設定為使用 C# 7.1 來進行編譯，否則上面代碼可能會造成編譯錯誤，錯誤訊息為 : Feature 'default literal' is not available in C# 7.0. Please use language version 7.1 or greater.，解決方法只要去設定專案編譯 C# 版本即可，設定方式為 專案點選右鍵 > property > Advance 

![](https://1.bp.blogspot.com/-r1oLSUnzP6c/XS0Cnd119vI/AAAAAAAAHP8/PwjTn29pjaEraXEUm9vun1RjQfFTjUhsACLcBGAs/s1600/visualstudioselectcharplanguage.png)

完成了基本功能設定，按下執行可以看到 Console 跑出來的結果如下

![](https://1.bp.blogspot.com/-NqpkaFDgGrc/XTKKsXRl76I/AAAAAAAAHQ0/48I8Wwcq80YyMp9thankcKMO8Z1aTtbEwCLcBGAs/s1600/GerericServer_timehosedservice_run.png)

這一篇介紹 Generic Host 基本設定，其中包含應用程式起來時候的配置，為了避免一篇文章太冗長看了想睡覺，下一篇再來介紹剩下的部分也就是 Quartz.Net 的方式，也可以趁這機會稍微消化一下，看是否有不懂的地方也歡迎隨時提出來，謝謝