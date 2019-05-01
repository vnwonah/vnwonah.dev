---
title: Running an ASP.NET Core Web Application on Alibaba Cloud Elastic Computing Service (ECS)
toc: true
#thumbnail: https://media.rehansaeed.com/rehansaeed/2014/02/NET-1024x576.png
date: 2019-05-01 07:25:02
category: Web
tags:
    - alibaba cloud
    - asp.net core
    - dotnet core
    - web application
    - cloud computing
---

ASP.NET Core is a Microsoft Web Framework for developing Web Applications that can run in any environment, including Windows and Linux servers. According to Microsoft Docs:

>ASP.NET Core is a cross-platform, high-performance, open-source framework for building modern, cloud-based, Internet-connected applications.

This tutorial provides detailed step by step instructions for a first time Alibaba Cloud user to create a Linux Elastic Compute Service (ECS) instance and host an ASP.NET Core Application on it.

## Why Alibaba Cloud ECS?
* * *

Alibaba Cloud Elastic Compute Service (ECS) provides fast memory and the latest Intel CPUs to help you to power your cloud applications and achieve faster results with low latency. All ECS instances come with Anti-DDoS protection to safeguard your data and applications from DDoS and Trojan attacks.

For this tutorial, our setup uses the Nginx Web Server to forward requests to a running ASP.NET core application. We will follow the under-listed steps to successfully create and deploy an ASP.NET Core Web Application to Alibaba Cloud ECS.

+ Create new Alibaba Cloud ECS Instance.
+ Create a new .Net Core Web Application using Microsoft Visual Studio.
+ Log into ECS Instance.
+ Install .NET Core Runtime on ECS.
+ Deploy Source Code to ECS Instance.
+ Configure Nginx.
+ Run Application.

You should have a valid Alibaba Cloud account. If you don't have one already, sign up to the Free Trial to enjoy up to $300 worth in Alibaba Cloud products. To create a new Alibaba Cloud ECS instance, you can follow the steps in this tutorial. You can also refer to this tutorial to set up your first Ubuntu server on Alibaba Cloud.

## Create a New .NET Core Web Application using Microsoft Visual Studio
* * *

We assume that you already have Visual Studio installed on your local machine. If you want to skip this step you can use a repo I already created at https://github.com/vnwonah/AlibabaCloudECSDeploy

* Open Visual Studio. Go to File > New Project. Under the C# node, select Web, then select ASP.NET Core Web Application. Give your application any name. (Do not include spaces in name).
* Click Ok, then select Web Application (Model – View – Controller). Leave Authentication to No Authentication. Click Okay.
* Run the Application and confirm that it runs on localhost in your browser.
Commit your code and push to GitHub (or any Repository Service).

## Log into ECS Instance
* * *

I will be using Ubuntu on Windows to SSH into the Instance. To learn how to install any Linux distro on Windows see article here.

* Go to the console, Click on your ECS Instance.
* Under Configuration Information, copy the Internet IP Address. It should look this this: 47.89.106.74
* Open your terminal and type ssh root@YOUR_IP_ADDRESS. (Replace YOUR_IP_ADDRESS with the Instance Internet IP Address.
* You will see a warning, answer yes. Enter your set password. You should now see your ECS Welcome Message.

## Install .NET Core Runtime on ECS
* * *

To run our .NET Web Application on Alibaba Cloud we need to install the .NET Core Runtime. Use the following commands:

{% codeblock lang:bash %}
wget -q https://packages.microsoft.com/config/ubuntu/16.04/packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
sudo apt-get install apt-transport-https
sudo apt-get update
sudo apt-get install dotnet-sdk-2.1
{% endcodeblock %}
    

Type <b>dotnet –version</b> to confirm that the .NET Core Runtime is installed.

## Deploy Source Code to ECS Instance
* * *

Now we will pull our source code into the ECS Instance. For this tutorial we will simply pull into our user folder.

Install git on the ECS instance. First run apt-get update, then run the command apt install git. Confirm that git installed successfully by typing <b>git –version</b> and see the output.

Install libunwind08 with the command: 

{% codeblock lang:bash %}
sudo apt-get install libunwind8
{% endcodeblock %}
    
Clone your repository using: 

{% codeblock lang:bash %}
git clone https://github.com/vnwonah/AlibabaCloudECSDeploy
{% endcodeblock %}
   
CD into the directory containing the .csproj file. In our case we use the command: 

{% codeblock lang:bash %}
cd AlibabaCloudECSDeploy/AlibabaCloudECSDeploy
{% endcodeblock %}
    
Then type the command:

{% codeblock lang:bash %}
dotnet publish -c Release -o ./published -r linux-x64
{% endcodeblock %}

This restores all dependencies for our project and builds a .dll file. At this point we are ready to configure Nginx and use it to forward requests to our application.

## Configure Nginx
* * *

In this section, we will install and configure Nginx to proxy requests to our .NET Core Web Application.

Install Nginx with the command sudo apt-get install nginx. This command installs the Nginx Web Server. At this point if you open your ECS Public IP Address in a browser, you should see the Nginx Welcome Page.

We need to configure Nginx to forward requests to the .NET Core Application. To do this, run the command <b>sudo pico /etc/nginx/sites-enabled/default</b>, clear everything in the file and type in

{% codeblock lang:bash %}
[Unit] 
Description=Alibaba Cloud Net Core App
[Service] 
WorkingDirectory=/root/netcoreapp
ExecStart=/usr/bin/dotnet /root/AlibabaCloudECSDeploy/ALibabaCloudECSDeploy/AppName.dll
Restart=always 
RestartSec=10 # Restart service after 10 seconds if dotnet service crashes 
SyslogIdentifier=dotnet-core-app
User=root
Environment=ASPNETCORE_ENVIRONMENT=Production  
[Install] 
WantedBy=multi-user.target
{% endcodeblock %}

Now enable the service using the command <b>sudo systemctl enable dotnet-core-app.service and start it using sudo systemctl start dotnet-core-app.service</b>.

At this point you should open a browser and navigate to your ECS Public IP Address and you will see your ASP.NET Core Application now running.

## Conclusion
* * *

In this article we went from creating a new Alibaba Cloud Elastic Compute Service (ECS) Instance to hosting a .NET Core Web Application using Nginx as Server on it.


