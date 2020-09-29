**前言**
在 ASP.NET 中常見的排程框架不外乎 Quartz.NET 與 Hangfire 兩種，過去自己在開發上比較常用到 Hangfire 搭配其後台管理介面，在使用上可以說是相當方便與容易上手，最近在新專案也有遇到 schedule 的需求，同事大推 Quartz.Net 來擔任工作排程器的工作，Quartz.Net 是一套功能齊全的工作排程框架，由 Java 熱門的排程框架 [Quartz](http://www.quartz-scheduler.org/) 移植到 .NET 上，open source 且提供彈性的設定讓開發者使用，在新版 3.0.7 支援 .NET Core 2.1 版本，今天就來簡單介紹 Quartz.NET 的安裝與基本應用使用，若有問題或是錯誤的地方歡迎各位高手給予指導。



**安裝**
首先，先建立一個名稱為 QuartzNetConsole 的 Console 專案，接著開啟 Nuget Package Mnage 輸入 "quartz" 搜尋，安裝目前最新版的 Quartz.NET 套件

![](https://2.bp.blogspot.com/-FQRUb4oNRVk/XL3YMRBweMI/AAAAAAAAGs4/33NGEMx52S4dPW64DYSKTVXbjBoQL9XkwCLcBGAs/s1600/QuartzNetInstall.png)

或是在 Nuget Package Console 輸入指令

```powershell
Install-Package Quartz 
```

如果有 Json 序列化需求，也可以一併加入 Quartz.Serialization.Json

**使用**
在使用前先介紹 Quartz.Net 中的幾個重要 API 與 Interface



- IScheduler : 主要工作排程 API、透過 Start 方法 run 排程。

- JobBuilder、IJobDetail : 透過 JobBuilder.Create 產生 IJobDetail 的 Instance。 

- TriggerBuilder、ITrigger : 透過 TriggerBuilder.Create 產生 ITrigger 的 Instance。  

IJob : 自定義的排程類別要實作的 Interface

簡單整理關係圖如下

![](https://2.bp.blogspot.com/-zeebcnebRzo/XL3rqcqKl6I/AAAAAAAAGtE/NA3IWxCDRocxrMn3vP6ToexGIStRU_6qwCLcBGAs/s1600/QuartzNetAPIInterface.png)

接著在 Console 專案 Program.cs 中的 Main 加入下列代碼

```C#
class Program
{
    static void Main(string[] args)
    {
        // trigger async evaluation
        RunProgram().GetAwaiter().GetResult();
    }
 
    private static async Task RunProgram()
    {
        try
        {
            // 建立 scheduler
            StdSchedulerFactory factory = new StdSchedulerFactory();
            IScheduler scheduler = await factory.GetScheduler();
 
            // 建立 Job
            IJobDetail job = JobBuilder.Create<ShowDataTimeJob>()
                .WithIdentity("job1", "group1")
                .Build();
 
            // 建立 Trigger，每秒跑一次
            ITrigger trigger = TriggerBuilder.Create()
                .WithIdentity("trigger1", "group1")
                .StartNow()
                .WithSimpleSchedule(x => x
                    .WithIntervalInSeconds(1)
                    .RepeatForever())
                .Build();
 
            // 加入 ScheduleJob 中
            await scheduler.ScheduleJob(job, trigger);
 
            // 啟動
            await scheduler.Start();
 
            // 執行 10 秒
            await Task.Delay(TimeSpan.FromSeconds(10));
 
            // say goodbye
            await scheduler.Shutdown();
        }
        catch (SchedulerException se)
        {
            await Console.Error.WriteLineAsync(se.ToString());
        }
    }
}
```
其中要執行的 ShowDataTimeJob 類別代碼如下

```C#
internal class ShowDataTimeJob :IJob
{
		public async Task Execute(IJobExecutionContext context)
    {
        await Console.Out.WriteLineAsync($"現在時間 {DateTime.Now}");
    }
}
```
程式說明



- 建立 scheduler : 透過 factory.GetScheduler() 取得 schedule

- 建立 Job : 使用 JobBuilder.Create 建立 ShowDataTimeJob，並定義其 key 與 group 名稱

- 建立 Trigger : 使用 TriggerBuilder 建立 ITrigger ，定義其 key 與 group 名稱，並設置立即執行執行時間為每一秒執行一次，其中 ShowDataTimeJob 的 Execute 就是此執行。執行結果如下:

![](https://4.bp.blogspot.com/-jlvMLr8K-ZE/XL3x-z2lzbI/AAAAAAAAGtQ/YO4bPy2afDckerovMMIBe8Bzlnjq6d0OgCLcBGAs/s1600/quartzNETresult.gif)

**感想**
上透過簡單的說明與操作，就可以快速的建立出 Quartz.NET 工作排程的功能，但其實在現實生活中的需求及實現的代碼往往都不會那麼簡單，如果有興趣可以先參考官方網站的開發說明文件，日後如果遇到在分享給各位，Happy Coding :)。

另外，Quartz.Net 也能搭配其他套件將設定檔設定 XML 檔中，甚至也能將夠搭配一點手做精神，手刻一些代碼後將設定檔放置在 YAML 檔中，方便配置。此外，JOB 的定義及執行的週期也能夠透過安裝其它套件，將其存放在 SQL Server 或是 Redis 中，得到更彈性的配置。又或者，能夠像設定檔一下放置在 YAML 檔案中。