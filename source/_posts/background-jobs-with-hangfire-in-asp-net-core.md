---
title: Background Jobs with Hangfire in ASP.Net Core
toc: true
date: 2019-11-03 10:53:51
category: Web
tags:
    - asp.net core
    - dotnet core
    - web application
    - web development
    - hangfire
    - background jobs
    - architecture
---

This article discusses background jobs; what they are, why use them and how to set them up in an ASP.Net Core Web Application. There are several options for implementing background jobs but the framework of choice here is hangfire because of its ease of entry for beginners.

## Takeaway

***

The takeaways from this article are:

1. What background jobs are.
2. What situation requires Background Jobs.
3. How to use Hangfire to create background jobs in ASP.Net Core.

<!--More-->

## What are Background Jobs

Background Jobs are processes that run behind the scenes and require very little or no interaction at all. A typical example of a process that can be a background job is sending monthly account statements to account holders in a bank. The steps for this process every month will be:  

1. Get opening balance for month.
2. Get list of credits and debits for month.
3. Calculate closing balnce for month.
4. Create pdf file with details.
5. Get customer email address.
6. Send email with pdf file to customer.

This process is fairly straightforward and will happen every month. It can be scheduled to happen every 30th day of the month at midnight, and can start automatically with no one having to trigger it. This kind of operation that will start automatically and require no human interaction to start is a recurrent background job.  

## What situations requires Background Jobs

Typically, user triggered operations that do not need to return a result to the user immediately are usually good candidates for background jobs. Imagine we process results for 5000 students. We have their exam records in an excel file; this file will be uploaded to our website, we will get their names and score, calculate the average for each student and store this information in a database.

If we approach this problem without using background jobs, we may want to simply do all the processing in a loop after getting the request from our controloler. The problem with this aproach is that the process will take two long, and time out after 5 minutes. (your browser will show a "request timed out message" to the user.) This isn't the behavior we want.

An appropriate way to handle this would be to start processing the student results as background jobs, and then return a message to the user immediately saying the results have been queued for processing. The advantages of this approach are numerous but two immediately obvious ones are a guarantee that all our results will be processed (no timeouts), and a better user experience for our user (no long wait time). 

## Create a Background Job with Hangfire in ASP.Net Core

We will create an ASP.Net Core Web Application in this section. This Application will enable a user create a Single Todo Item. 

We will extend the application to accept a CSV (or Microsoft Excel) file containg a list of todo items. This items will be added to the application using background jobs. You can follow along with the steps below or get the complete project on github at : 

1. Create an ASP.Net Core 2.2 Web Application (Model-View-Controller). Uncheck Enable Docker Support.
2. Add a new Class Todo in the models folder. 
{% codeblock lang:C#%}
public class Todo
{
    public string Text { get; set; }
    public DateTime DueDate { get; set; }
    public bool Completed { get; set; }
}
{% endcodeblock %}

3. Create a folder Data in solution explorer. Add a Database Context Class AppDbContext.cs in this folder.
{% codeblock lang:C#%}
using Microsoft.EntityFrameworkCore;
namespace AspNetCoreHangfire.Data
{
    public class AppDbContext : DbContext
    {
        public AppDbContext(DbContextOptions<AppDbContext> options)
          : base(options)
        {

        }
    }
}
{% endcodeblock %}

4. Goto Startup.cs add the using statement:
{% codeblock lang:C#%}
    using Microsoft.EntityFrameworkCore;
{% endcodeblock %}
scroll to the method **ConfigureServices** and add the line below to register the AppDbContext
{% codeblock lang:C# %}
    services.AddDbContext<AppDbContext>(options => options.UseInMemoryDatabase("AppDb"));
{% endcodeblock %}

5. Right click on the controllers folder in solution explorer and click add new controller. Select MVC Controller with views, using Entity Framework, Click Add.
![Right-click Controllers folder and click Add Controller](https://res.cloudinary.com/vnwonah/image/upload/v1572828253/add_controller_1_gevhjs.png)
![ Select MVC Controller with views, using Entity Framework](https://res.cloudinary.com/vnwonah/image/upload/v1572828254/add_controller_2_f7fujs.png)

6. In the dialog, Select the Todo model and AppDbContext we created, click Add. Allow Visual Studio generate the necessary pages
![Select the Todo model and AppDbContext we created, click Add](https://res.cloudinary.com/vnwonah/image/upload/v1572829328/add_controller_3_kx8dlq.png)
We can go ahead now to run our application and add some items to our Todo List. At this point we are ready to upload add Hangfire to our application.
![Add a new Todo Item](https://res.cloudinary.com/vnwonah/image/upload/v1572833679/Index-AspNetCoreHangfire-Persona_hxtgmw.gif)

7. Add the following nuget packages to the Application
    * Hangfire.Core
    * Hangfire.SqlServer
    * Hangfire.AspNetCore
    * Hangfire.MemoryStorage

8. Open Startup.cs and add the following using statements
{% codeblock lang:C# %}
using Hangfire;
using Hangfire.MemoryStorage;
{% endcodeblock %}
next, add the following lines to **ConfigureServices** to setup Hangfire
{% codeblock lang:C# %}
services.AddHangfire(configuration => configuration
.SetDataCompatibilityLevel(CompatibilityLevel.Version_170)
.UseSimpleAssemblyNameTypeSerializer()
.UseRecommendedSerializerSettings()
.UseMemoryStorage()  //using inmemory storage. comment this and uncomment below to use an actual database
//.UseSqlServerStorage(Configuration.GetConnectionString("HangfireConnection"), new SqlServerStorageOptions
//{
//    CommandBatchMaxTimeout = TimeSpan.FromMinutes(5),
//    SlidingInvisibilityTimeout = TimeSpan.FromMinutes(5),
//    QueuePollInterval = TimeSpan.Zero,
//    UseRecommendedIsolationLevel = true,
//    UsePageLocksOnDequeue = true,
//    DisableGlobalLocks = true
//})
);
{% endcodeblock %}
finally goto the **Configure** method and add
{% codeblock lang:C# %}
app.UseHangfireDashboard();
{% endcodeblock %}

9. Run the application and navigate to https://localhost:44338/hangfire/jobs/enqueued to see the hangfire dashboard
![Hangfire Dashboard](https://res.cloudinary.com/vnwonah/image/upload/v1572832592/hangfire_enqueued_xgitcq.png)
We currently have no sceduled or complete jobs.

10. Next is to create a few jobs. For this we will upload a list of todo Items in a csv file and use a background job to add them to the Database.

