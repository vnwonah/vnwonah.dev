---
title: Dependency Injection in Asp.Net Core for Beginners - Part 1
toc: true
date: 2019-07-09 13:53:00
category: Web
tags:
    - asp.net core
    - dotnet core
    - web application
    - web development
    - dependency injection
---

There are a lot of concepts that beginner programmers encounter when they start to go beyond writing simple console applications in dotnet and maybe try to build a website. One of such concepts is Dependency Injection (DI).

## Takeaway

***

After reading this article you will know the following:

1. What dependency Injection is.
2. Why we need dependency Injection.
3. How we use/do dependency injection in dotnet core.

<!--More-->

## Prerequisite Knowledge

***

It would be nice that you know what objects are in C#, how we define and instantiate them, and the differnce between one instance of an object and the other. However, if you don't, reading through might give you an idea.

## Note

***

This article is a rather high level overview to get beginners started with Dependency Injection.  If you feel that cruicial details are left out, this is probably intentional, and I probably do not think a complete beginner to the subject needs those details.

## Dependency Injection

***

While I could rattle out a Computer Science definition for DI, I would prefer to show you what it is. Let us build an application that doesn’t use DI see the problems with it, then change it to use DI

Assume you have a Website that lets people send Coffee to their email, (sorry guys, this is the best example I could come up with, not very creative, but i will explain).

Back to drinking Coffee, Our web app allows a user to browse to a page, enter their email and send themselves a cup of cofee. This may seem abstract (or not), but let us look at the steps involved in sending Coffee.

1. First we need to get a cup from a waiter
2. Then we need the waiter to pour some Coffee into our cup
3. Then we send the cup of Coffee to our email.
4. Finally we can enjoy (or not) some delicious black Coffee.
**THE END**

Actually no, now we know the process involved in getting Coffee to our user. We have an algorithm, (otherwise known as step by step instructions to solve a problem). Now we must convert our steps into code; line by line. See what I did there? step by step, line by line. Hehe.

Anyways, lets do this **without** Dependency Injection first.

Lets create an asp.net core web application (this can be either asp.net core 1.0 or 2.2) using the model view controller template and no authentication. call it **CoffeeGiver**

Go ahead and fire up Visual Studio and create it.

We need a cup to put Coffee in

In the Models folder, Create a new Class named Cup like below

{% codeblock lang:C# %}
public class Cup
{
   public bool IsFilled { get; set; } = false;
}
{% endcodeblock %}

every new cup we get will not be filled by default. Next, let us create a waiter that will give us a new cup, and fill it with Coffee.

{% codeblock lang:C# %}
public class Waiter
{
    public Cup ProvideCup()
    {
        //gives us a new cup
        return new Cup();
    }

    public void FillCup(Cup cup)
    {
        //fills the cup
        cup.IsFilled = true;
    }
}
{% endcodeblock %}

The waiter has two methods (or functions), one gives us a cup, while the other takes the cup and fills it with Coffee. Next we go to our Home Controller and change Index Action to below

{% codeblock lang:C# %}
public IActionResult Index(string email = null)
{
    if (string.IsNullOrWhiteSpace(email))
    {
        return View();
    }
    else
    {
        //we have a new waiter to serve our request
        Waiter waiter = new Waiter();

        //we have a new cup from our new waiter
        Cup newCup = waiter.ProvideCup();

        //our waiter now fills the new cup with Coffee
        waiter.FillCup(newCup);

        //send filled cup to users email 
        return Content($"Yay! Coffee sent to {email}");
    }
}
{% endcodeblock %}

Next, go to the view in Views/Home/Index.cshtml, clear everything and add this code snippet:

{% codeblock lang:Html %}
@{
    ViewData["Title"] = "Home Page";
}

<form method="get">
    <input name="email" type="email"/>
    <button>Send Coffee!</button>
</form>

{% endcodeblock %}

next run the solution, fill in your email and send yourself Coffee!

![Request Coffee Web Page](https://res.cloudinary.com/vnwonah/image/upload/v1562680276/coffeegiver1_rvz59y.png)
![Success Web Page](https://res.cloudinary.com/vnwonah/image/upload/v1562680275/coffeegiver2_arqwwu.png)

This works perfect but there is a fatal flaw in our solution. The offending line of code is:

    Waiter waiter = new Waiter();

**Why?**

For every request for a cup of Coffee, we are creating a new Waiter. This is like hiring a new waiter for everyone that comes to eat at your restaurant. This is simply impractical in the real world, and shouldn't be done in computer programs too. (unless explicitly needed.)

What if our program has one waiter that's just there waiting, and takes requests as they come in? as opposed to creating a new one everytime? That sounds great. While there is a number of ways to do this, let us use, you guessed it; **Dependency Injection ༼ つ ◕_◕ ༽つ**

Goto your Startup.cs file and add the following line of code.

    services.AddSingleton<Waiter>();

What does it do?

When our application is first run; say after deploying it to azure or some other host, this line of code creates a single waiter, and can give this waiter to any part of our code that depends on it.

A techy way to say this is that our application can "inject" this waiter to any part of our code that "Depends" on the waiter to work.
Boom: Dependency Injection. Let us now change our code in our controller to use this single waiter.

Add the field

    private Waiter _waiter; to the home controller class.

Next create a constructor for your HomeController Acceping a waiter as argument

{% codeblock lang:C# %}
public HomeController(Waiter waiter)
{
    _waiter = waiter;
}
{% endcodeblock %}

Now that we have the waiter available in our controller, we do not need to create one when a request comes in. Change the code in the else block of our Index Action to

{% codeblock lang:C# %}

//we are getting a new cup from our injected waiter
Cup newCup = _waiter.ProvideCup();

//our injecter waiter now fills the new cup with Coffee
_waiter.FillCup(newCup);

//send filled cup to users email
return Content($"Yay! Coffee sent to {email}");
{% endcodeblock %}

With these changes, our solution still works as supposed, but we have fixed the flaw using DI.

So now,

### What is Dependency Injection

***

Dependency Injection is a concept that allows you give certain parts of code the resources they need, without having to create them in the class. Note that it also allows sharing of the same resources across multiple parts of code. (for example we could inject the same waiter into another controller).

### Why do we need Dependency Injection

***

There are often objects that we need shared, or created just once, in our application, these are perfect candidates for DI. Also using DI is often cleaner.

### How do we use DI in dotnet core

***

To use DI in dotnet core, first we need to "register" the component you need injected in startup.cs

    services.AddSingleton<Waiter>();

now you can just inject the component into your constructors, save to a field and use

{% codeblock lang:C#%}
private Waiter _waiter;

public HomeController(Waiter waiter)
{
    _waiter = waiter;
}
{% endcodeblock %}

I hope you understand Dependency Injection to some extent now. Please note that you can use DI to inject new instances each time a dependency is needed. This article is intentionally limited for the complete beginner.

You can find the code for this example [link](https://github.com/vnwonah/Coffee-giver "on github"). Please leave questions/feedbacks. I really appreciate the feedback. Thanks for reading.
