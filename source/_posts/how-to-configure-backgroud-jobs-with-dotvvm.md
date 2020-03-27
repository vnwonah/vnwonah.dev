---
title: How to Configure Background Jobs with Dotvvm
date: 2020-03-27 11:33:53
category: Web
tags:
    - dotvvm
    - dotnet core
    - hangfire
    - web development
    - queueing
    - background jobs
---


In this post we have talked about how to configure and run bckground jobs in a Dotvvm Application. Some of the questions you may ask yourself when you decide to use Dotvvm is if you can use the same frameworks and tools you're used to when you're using Asp.NetCore, the answer is yes, and this post shows you how to use hangfire in Dotvvm.

## Introduction

Most Web Application/Web Sites follow a simple request/response pattern; the user makes a request to a url and a view is returned to their browser, or they send information through input fields, some processing is done, and an acknowledgement is sent back to them with the status of the operation performed. This pattern works for most processes that will happen on a web application, but there are certain situations where this simple Request - Process - Response pattern doesn't work or aren‘t suitable for.

<!--More-->

There are operations a user will require from a web application that cannot return an immediate response, there can be a number of reasons, the most common being that it will simply take too long to complete the operation; and it is impractical to have the user wait for a response immediately, or there just isn't enough resources to give a response to every user requesting for that operation at the same time.

An example of one of these kind of operation is requesting for your account statements on your online banking platform. Simple as this may seem, it may potentially cost a lot in compute resources. A quick rundown of the things that possibly happen when you request your statement is;

1. All your transactions up to current date is retrieved from the database.
2. Calculations are run to make sure there are no mistakes/errors
3. The data is processed into a pdf file which can amount to several Megabytes.
4. The file is uploaded to storage and a link for download returned to you.

While it is possible to perform these operations and return the statement to the user in a response immediately they request it, it can become a bit of a performance hit when multiple people are requestiong for statements, and the web application is spending all of its compute resources servicing these requests. This can impact other parts of the system and lead to users noticing slow response times. 

The proper way to handle these compute intensive tasks is to offload them onto so called background jobs. The flow is relatively simple; the user initiates a potentially compute intensive operation (like generating their account statement from inception), we immediately return a response to them stating that they will recieve the statement in their email in X amount of time, we start processing the statement on a background thread, with just as much resources as we want to dedicate to it; on completion, an email with either a download link or the file itself is sent to the requesting user. The primary gain here is that this clears up our http request pipeline to accept new requests, It is also more reliable as the risk of our user losing connectivity and making another request is eliminated; with background jobs, we can simply check if we already have a job running for that user, and ignore new requests.

In the following Sections, We will build a WebApplication that adds students to a database using DotVVM Web Aplication Framework and Hangfire for background job processing.

For the benefit of those unfamiliar with DotVVM, it is a Mature Model-View-ViewModel Web Application Framework for dotnet, and a dotnet foundation project. It provides a way to build web applications using the MVVM pattern, an alternative approach to the Model-View-Controller pattern. One of the key benefits for using an MVVM framework is UI Test automation; since the ViewModel will most often have a 1:1 correspondence to the View, automating tests on the view model can be done much more easily than akwauard UI Interaction and automation. You can usbscribe to our blog here to be notified when the article highlighting key benefits of the MVVM Pattern is published.

## Adding DotVVM Project Templates to Visual Studio

To add DotVVM Project templates to Visual Studio, download and install DotVVM for Visual Studio 2019 or for Visual Studio 2017 from the Visual Studio Marketplace. The install adds three project templates

* DotVVM Web Application (OWIN on .Net Framework)
* DotVVM Web Application (.Net Core)
* DotVVM Web Application (ASP.NET Core on .Net Framework)

We will be using the DotVVM Web Application (ASP.NET Core on .Net Framework) template.

## Create a DotVVM Application

* Open up the Create New Project dialog in Visual Studio 2019, type in dotvvm in the search bar and select Dotvvm Web Application (.Net Core), then click Next.
* Enter a name for the project, you can use DotvvmHangfireDemo, click Create
* On the popup check Add Bootstrap 3, Add jQuery, and Sample Crud Page
* Change the .Net Core Version to version 2.2, Click Create/Ok

![Fig 1.0: Creating a new Dotvvm Web Application](https://res.cloudinary.com/vnwonah/image/upload/v1585309054/creating_a_new_dotvvm_application_bifucs.png)

At this point, build the new project and run it to make sure everything runs correctly, you should see an empty student's list table like the screen below, click new item and create a new student to confirm that everything works correctly.

This method adds one student at a time to the database, using HangFire, we will upload a csv file, extract a list of students from it and add tem to the database all in the background

## Adding and Configuring Hangfire

to use hanfire in our application, we need to add a few nuget packages and so some initial configuration, follow the steps below:

* Add the nuget packages Hangfire.AspNetCore, HangFire.Core and Hangfire.SqlServer
* Open Startup.cs and add the following using statements

{% codeblock lang:C# %}

using Hangfire;
using Hangfire.SqlServer;

{% endcodeblock %}

* Still in Startup.cs, add the code snippet below to your ConfigureServices method

{% codeblock lang:C# %}

services.AddHangfire(configuration => configuration
.SetDataCompatibilityLevel(CompatibilityLevel.Version_170)
.UseSimpleAssemblyNameTypeSerializer()
.UseRecommendedSerializerSettings()
//.UseMemoryStorage()
.UseSqlServerStorage(Configuration.GetConnectionString("DefaultConnection"), new SqlServerStorageOptions {
CommandBatchMaxTimeout = TimeSpan.FromMinutes(5),
SlidingInvisibilityTimeout = TimeSpan.FromMinutes(5),
                   QueuePollInterval = TimeSpan.Zero,
                   UseRecommendedIsolationLevel = true,
                   UsePageLocksOnDequeue = true,
                   DisableGlobalLocks = true
               })
           );
 services.AddHangfireServer();

{% endcodeblock %}

* Run the project and go to <http://localhost:you-port-number/hangfire> . if you have everything correctly configured you should see the hangfire dashboard like below.

![Configured Hangfire](https://res.cloudinary.com/vnwonah/image/upload/v1585312074/configured_hangfire_srf1ck.png)

* At this point we have hangfire setup and are ready to run backgroud jobs with it, we now need to add the views and logic to handle uploading csv file and saving students from that csv file to the database.

## Adding Upload/Process Students Functionality to our Service Class

In the Services/StudentService.cs file, add the two methods shown below; the first one retrieves a file from the uploaded files folder, gets the list of students from the file and writes them to the database.
The second one just checks the students in the database and remove duplicates. These methods do not return any data back to the User Interface, and depending on the size of data can run for a really long time; they are the perfect candidates for a background job.

{% codeblock lang:C# %}

public void ProcessUploadedFile(Guid fileId)
{
    StreamReader file = null;
    try
    {
        string line;
        file = new StreamReader(storage.GetFile(fileId));
        while ((line = file.ReadLine()) != null)
        {
            //we are getting eac part of the todo into an array.
            //parts[0] will be the Todo Text and parts[1] will be the Due date
            var parts = line.Split(",");
            //first part of this if statement checks that the Todo Text is a valid non empty text
            //second part checks that we are passing in a valid date time
            if (!string.IsNullOrWhiteSpace(parts?[0])
                && !string.IsNullOrWhiteSpace(parts?[1])
                && !string.IsNullOrWhiteSpace(parts?[3])
                && DateTime.TryParse(parts?[2], out DateTime enrollmentDate))
            {
                var model = new StudentDetailModel
                {
                    FirstName = parts?[0],
                    LastName = parts?[1],
                    EnrollmentDate = enrollmentDate,
                    About = parts?[3]
                };
                //this line creates a background job for each Todo Item
                BackgroundJob.Enqueue(() => InsertStudentAsync(model));
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
        storage.DeleteFile(fileId);
    }
}
public void DeleteDuplicateStudents()
{
    var duplicateStudents = studentDbContext.Students.GroupBy(s => new { s.FirstName, s.LastName }).SelectMany(s => s.OrderBy(y => y.Id).Skip(1));
    studentDbContext.Students.RemoveRange(duplicateStudents);
    studentDbContext.SaveChanges();
}

{% endcodeblock %}

## Adding Upload/Process Students Functionality UI pages

Dotvvm handles files uploads really well and we can map a temp folder where it keeps uploaded files, this functionality isn't enabled by default but can be enabled by going to the DotvvmStartup.cs file and adding services.AddDefaultTempStorage("temp"); Your ConfigureServices method in DotvvmStartup.cs should look like below

{% codeblock lang:C# %}

public void ConfigureServices(IDotvvmServiceCollection options)
{
    options.AddDefaultTempStorages("temp");
    options.AddUploadedFileStorage("App_Data/Temp");
}

{% endcodeblock %}

The csv file format used here can be downloaded  from <https://1drv.ms/u/s!AjoKAnDVPJCF-1gjJKfuvrmizJOS?e=Ym8Bhg>. Follow the steps below to add this upload/process functionality

* In the ViewModels/CRUD folder add a new C# file, name it BulkCreateViewModel.cs, add the code below to the file. The process file method takes our uploaded file and passes it to the StudentService for processing using Hangfire's BackgroundJob.Engueue, and returns a response to the user without waiting for the file processing to finish.

{% codeblock lang:C# %}

public class BulkCreateViewModel : MasterPageViewModel
{
    private readonly StudentService studentService;
    private IUploadedFileStorage storage;
    public bool CanProcess { get; set; }
    public string Message { get; set; }
    public UploadedFilesCollection Files { get; set; }
    public BulkCreateViewModel(
        StudentService studentService,
        IUploadedFileStorage storage)
    {
        this.studentService = studentService;
        this.storage = storage; 
        Files = new UploadedFilesCollection();
        CanProcess = false;
        Message = string.Empty;
    }
    public void ProcessFile()
    {
        // do what you have to do with the uploaded files
        CanProcess = true;
    }
    public void Process()
    {
        var uploadPath = GetUploadPath();
        // save all files to disk
        foreach (var file in Files.Files)
        {
            var targetPath = Path.Combine(uploadPath, file.FileId + ".csv");
            storage.SaveAs(file.FileId, targetPath);
            BackgroundJob.Enqueue(() => studentService.ProcessUploadedFile(file.FileId));
        }
        // clear the uploaded files collection so the user can continue with other files
        Files.Clear();
        CanProcess = false;
        Message = "Students are being inserted in the background";
    }
    private string GetUploadPath()
    {
        var uploadPath = Path.Combine(Context.Configuration.ApplicationPhysicalPath, "MyFiles");
        if (!Directory.Exists(uploadPath))
        {
            Directory.CreateDirectory(uploadPath);
        }
        return uploadPath;
    }
}

{% endcodeblock %}

* In the Views/CRUD folder add an item, name it BulkCreate.dothtml, and add the code below to the page

{% codeblock lang:dothtml %}

@viewModel DotvvmHangfireDemo.ViewModels.CRUD.BulkCreateViewModel, DotvvmHangfireDemo
@masterPage Views/MasterPage.dotmaster
@import DotvvmHangfireDemo.Resources

<dot:Content ContentPlaceHolderID="MainContent">
    <div class="page-center">
        <dot:RouteLink RouteName="Default" Text="Go back" class="page-button btn-back btn-long" />
        <div class="page-box">
            <h1>Bulk Create</h1>
            <div>
                <dot:FileUpload UploadedFiles="{value: Files}" AllowMultipleFiles="false" SuccessMessageText="File is ready for processing" UploadCompleted="{command: ProcessFile()}" />
            </div>
            <div>
                {% raw %}
                {{value: Message}}
                {% endraw %}
            </div>
            <div class="btn-container">
                <dot:Button Text="Add Students to Database" Click="{command: Process()}" Enabled="{value: CanProcess}"/>
            </div>
        </div>
    </div>
</dot:Content>

{% endcodeblock %}

* Open Views/Default.dothtml and paste the code snippet

{% codeblock lang:dothtml %}

<dot:RouteLink Text="Bulk Create" RouteName="CRUD_BulkCreate" class="page-button btn-long"/> under 

<dot:RouteLink Text="{resource: Texts.Label_NewStudent}" RouteName="CRUD_Create" class="page-button btn-right btn-long" />

{% endcodeblock %}

This adds a button that routes to our BulkCreate page.

* Run the project and click the Bulk Create Button. If everything works correctly, then we are ready to test

## Testing out the bulk upload functionality

To test it out, download a CSV file containing about 1000 students from <https://1drv.ms/u/s!AjoKAnDVPJCF-1gjJKfuvrmizJOS?e=Ym8Bhg>, some of them are duplicates.

* Run the project, navigate to the bulk create page, upload the file and click Add Students To Database

* Navigate to <http://localhost:your-port-number/hangfire/jobs/succeeded> and you should see a list of jobs that were executed as the students were added to the database.
![Jobs Completed](https://res.cloudinary.com/vnwonah/image/upload/v1585312289/jobs_completed_p8i1zv.png)

* Navigate to <https://localhost:your-port-number> and you should see the students added with duplicates.
![Duplicate Students](https://res.cloudinary.com/vnwonah/image/upload/v1585312369/duplicate_students_bgday3.png)

## Adding A Recurring Background Job to Clear Duplicates

So far, we have used hangfire to do one off job, or so called fire and forget jobs. But what if we want a job to run automatically witout input from a user say everyday at midnight? An example of such a task might be to clear the temporary uploads folder, or to remove duplicate data from the students database.

We already have the bit of code to removes duplicate student data from the database in our student service class, what's left is to tell hangfire to execute it automatically every x amount of time. This kind of background jobs are called Recurring Background Jobs.

To create a recurring background job in hangfire:

* Open Startup.cs and add the method below; the method simply gets an instance of studentServiceand and Creates a recurring Job using hangfire's RecurringJob.AddorUpdate method, passing in the DeleteDuplicateStudents method as the method to be executed periodically. It also specifies how often the job should run, in this case we are saying we want our Job to run daily, by calling using Cron.Daily().

{% codeblock lang:C# %}

public void CreateOrUpdateRecurringTasks(IServiceProvider serviceProvider)
{
    using(var scope = serviceProvider.GetRequiredService<IServiceScopeFactory>().CreateScope())
    {
        StudentService studentService = scope.ServiceProvider.GetRequiredService<StudentService>();
        RecurringJob.AddOrUpdate(() => studentService.DeleteDuplicateStudents(), Cron.Daily());
    }
}

{% endcodeblock %}

* Call this method from the configure services class as shown below

{% codeblock lang:C# %}

public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    // use DotVVM
    var dotvvmConfiguration = app.UseDotVVM<DotvvmStartup>(env.ContentRootPath);
    dotvvmConfiguration.AssertConfigurationIsValid(); 
    // use static files
    app.UseStaticFiles(new StaticFileOptions
    {
        FileProvider = new PhysicalFileProvider(env.WebRootPath)
    });
    app.UseHangfireDashboard();
    //Call recurring job method
    CreateOrUpdateRecurringTasks(app.ApplicationServices);
}

{% endcodeblock %}

* Run the application and Navigate to <http://localhost:your-port-number/hangfire/recurring>, you should see a new job with a Created date and a Bext execution Date. We can wait for the job to be triggered automatically in a day but hangfire also allows us to trigger the job now.
![Recurring Jobs](https://res.cloudinary.com/vnwonah/image/upload/v1585312434/recurring_jobs_r2hi2u.png)

* Select the Job and click trigger now then go to the jobs tab and verify that te Job Succeeded.

* Navigate back to the Home Page and verify that there is no longer any duplicate records.

## Conclusion

In this post we have talked about how to configure and run bckground jobs in a Dotvvm Application. Some of the questions you may ask yourself when you decide to use dotvvm is if you can use the same frameworks and tools you're used to when you're using Asp.NetCore, then answer is yes, and this post shows you how to use hangfire in Dotvvm.

The completed project can be downloaded from <https://github.com/vnwonah/dotvvm-hangfire-demo>