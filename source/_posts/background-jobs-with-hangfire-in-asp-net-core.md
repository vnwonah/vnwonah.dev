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

We will create an ASP.Net Core Web Application in this section. This Application will enable a user create a Single Todo Item. Link to sample application is provided at end.

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

10. Next is to create a few jobs. For this we will upload a list of todo Items in a csv file and use a background job to add them to the Database. Right lclick on the project in solution explorer and add a new folder, call it Services, then add a class called TodoService to this folder. Add the code below to the class;
{% codeblock lang:C# %}
using Hangfire;
using System;
using System.IO;

public class TodoService
{
    private AppDbContext _context;
    public TodoService(AppDbContext context)
    {
        _context = context;
    }
    public void ProcessUploadedFile(string filePath)
    {
        if (string.IsNullOrEmpty(filePath))
            throw new ArgumentException();
        //we are throwing an argument if an invalid filepath is given to us to process
        StreamReader file = null;
        try
        {
            string line;
            file = new StreamReader(filePath);
            while ((line = file.ReadLine()) != null)
            {
                //we are getting eac part of the todo into an array. 
                //parts[0] will be the Todo Text and parts[1] will be the Due date
                var parts = line.Split(",");
                //first part of this if statement checks that the Todo Text is a valid non empty text
                //second part checks that we are passing in a valid date time
                if(!string.IsNullOrWhiteSpace(parts?[0]) && DateTime.TryParse(parts?[1], out DateTime date))
                {
                    //this line creates a background job for each Todo Item
                    BackgroundJob.Enqueue(() => SaveNewTodoItem(parts[0], date));
                }
            }
            file.Close();
        }
        catch (Exception e)
        {
        }
        finally
        {
            if (file is object)
                file.Close();
            //delete the file
            if (File.Exists(filePath))
                File.Delete(filePath);
        }
    }
    public async Task SaveNewTodoItem(string text, DateTime dueDate)
    {
        var todo = new Todo
        {
            Text = text,
            DueDate = dueDate
        };
        await _context.AddAsync(todo);
        await _context.SaveChangesAsync();
    }
}
{% endcodeblock %}
11. The class we added above reads a file to the www directory of our website, picks out individual todo items from the lines in the file and uses a background job to add them to the database. We have just one bit left, how to upload file this class is going to read from.
12. Goto the TodosController class and add the following using statements
{% codeblock lang:C# %}
using System.IO;
using Hangfire;
{% endcodeblock %}
We need to inject the TodoService into the controller
{% codeblock lang:C# %}
private readonly AppDbContext _context;
private readonly TodoService _todoService
public TodosController(
    AppDbContext context,
    TodoService todoService)
{
    _context = context;
    _todoService = todoService;
}
{% endcodeblock %}
Now we need to add the action that will recieve the uploaded file, add the action below to the controller
{% codeblock lang:C# %}
[HttpPost]
public async Task<IActionResult> UploadFile(IFormFile file)
{
    try
    {
        if (file == null || file.Length == 0)
        {
            TempData["error"] = "File not Selected";
            return RedirectToAction("Index");
        }
        var path = Path.Combine(
                    Directory.GetCurrentDirectory(), "wwwroot",
                    file.FileName)
        using (var stream = new FileStream(path, FileMode.Create))
        {
            await file.CopyToAsync(stream);
        }
        BackgroundJob.Enqueue(() => _todoService.ProcessUploadedFile(path));
        TempData["success"] = "The Todo Items are being created"
    }
    catch (Exception)
    {
        TempData["error"] = "An Error occured, please retry";
    }
    return RedirectToAction("Index");
}
{% endcodeblock %}
Replace the Index Action with the code below
{% codeblock lang:C# %}
public async Task<IActionResult> Index()
{
    if(TempData["success"] is object)
    {
        ViewBag.Success = TempData["success"];
    }
    else if(TempData["error"] is object)
    {
        ViewBag.Error = TempData["error"];
    }
    TempData.Clear();
    return View(await _context.Todo.ToListAsync());
}
{% endcodeblock %}
Finally we will modify our views to handle the file upload process. Open the Views Folder, Then Todo and the Index.cshtml file. Add the foloiwng line before **Create New**
{% codeblock lang:HTML %}
@if (ViewBag.Success is object)
{
    <div class="alert alert-success" role="alert">
        @ViewBag.Success
    </div>
}
else if (ViewBag.Error is object)
{
    <div class="alert alert-danger" role="alert">
        @ViewBag.Error
    </div>
}
<row>
    <div class="col-md-12">
        <form method="post" enctype="multipart/form-data" asp-controller="Todos" asp-action="UploadFile">
            <div class="form-group">
                <div class="col-md-10">
                    <p>Upload one or more files using this form:</p>
                    <input type="file" name="file" />
                    <input type="submit" value="Upload" />
                </div>
            </div>
        </form>
    </div>
</row>
{% endcodeblock %}

13.  Now run the application and upload a file, you can dowload the sample csv from [here](https://eumcpq.ch.files.1drv.com/y4mpux2_ku8hcP8IkuEZ7yd6MAsLg0jF_pJBNM8NxWJluLSfeTv8diRTHrZrC9G5mbwMHc5E2MNkXYM6035H_B-QPFDAUb7_iI2ZPw-s4994eBF4B5lho9ElGq4pTSW3j6vtLqDibzVf5FeJitFaLln2-gBkWGI6QODjiXF-d7XbkVgifBnU58BdPMXX4ujDbIZOxF-iudsPDDnLVcHDcA-FAMgiwT-r6Pmonx2g__mj2Y/TodoList.csv?download&psid=1). After the page refreshes navigate to https://localhost:5001/hangfire/jobs/succeeded (remember to replace 5001 with your own port number) and you will see about 1055 completed jobs. You can click on any of those jobs to see details.
14.  Navigate back to  https://localhost:5001/todos and you should see anout 1055 created todo items, all saved to the database.

## Conclusion

We now have hangfire implemented in our application. You can download the sample application [here](https://github.com/vnwonah/ASPNetCoreHangfire) In this case we are processing just 1000 items but in a real scenario it could be several hundred thousands, background jobs are a powerful concept for such scenarios. If you have any questions or comments leave a comment or reach my email vnwonah@outlook.com.

If you enjoy my articles you can also buy me a coffee via Paypal!
