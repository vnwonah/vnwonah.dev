---
title: Json Property Names in Dotnet Core 3.x
toc: true
date: 2020-02-05 11:20:22
category: Web
tags:
    - asp.net core
    - dotnet core
    - web application
    - web development
    - asp.net core 3.0
    - model binding
---

## TL;DR

Pre-Dotnet Core 3.0 uses the format below to map incoming json properties to C# model properties

{% codeblock lang:C# %}
public class Model
{
    [JsonProperty("first_name")]
    public string FirstName

}
{% endcodeblock %}

Dotnet Core 3.x now uses

{% codeblock lang:C# %}
public class Model
{
    [JsonPropertyName("first_name")]
    public string FirstName

}
{% endcodeblock %}

Attempting to use the first version in a Dotnet Core 3.x Application will fail to bind and those properties will be null.

<!--More-->

## What Changed

Newtonsoft.Json, the Json Library has been removed from the ASP.Net Core Shared Framework in favour or the new System.Text.Json library. Since the JsonProperty attribute is from the Newtonsoft.Json library, removing it from ASP.Net Core means it can no longer bind property names. 

The new added Json API's is claimed to specifically designed for high-performance scenarios so defintiely check it out.
