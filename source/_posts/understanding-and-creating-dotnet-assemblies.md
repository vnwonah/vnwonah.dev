---
title: Understanding & Creating Dotnet Assemblies
date: 2019-11-07 13:27:23
category: System
tags:
    - dotnet
    - dotnet core
    - nuget
    - assemblies
---

If you've been doing C# development for any amount of time, chances are you've seen the term "Assembly" or "Assemblies" thrown around pretty often. This article helps the complete beginner understand what Assemblies are, how to use them, and how to make them.

## Takeaway

***

The takeaways from this article are:

1. Understand what an Assembly is
2. Understand why they are useful.
3. Learn how to make one.

<!--More-->

## What is an Assembly

For simplicity, think of an Assembly as a collection of C# classes that are bundled together,  except you are not moving the source files, but a compiled version of them. The compiled version will have a filename ending with .dll or .exe.

To elaborate, assume we've written a class for displaying some words to screen, and a second for sending to a printer. We happily use these classes in our current project. What do we do when we want to use the same bits of code in other projects?

The obvious solution is to recreate the classes in a new project, copy over the old code, and we're good to go. This works, except each time we add a new printing function to our class, we will need to open up all the projects we copied it into, and edit them individually, not convenient at all. We may also want to share our functionality with others, but not want them to see how we built it/see our code; we just want them to use it.

The situation described above is a perfect candidate for an assembly.  We can package our code into an assembly for use in other projects and share it with others. Enough talk, let's create a dll!

## Creating a Printer Assembly

The steps listed below take you through creating a simple assembly and using it in another project.

1. Open Visual Studio and create a new project, the project template should be Class Library (.Net Core). Call this project UniversalPrinter, click Create.
2. Add a folder to the UniversalPrinter project (not the solution), name the folder Printers. It will hold all the different mediums our library can output text to.
3. Add two classes to the folder; PaperPrinter.cs and ScreenPrinter.cs.
{% codeblock lang:C# %}
using System;
namespace UniversalPrinter.Printers
{
    public class ScreenPrinter
    {
        public void Print(string text)
        {
            //Actual code for showing text on screen goes here
            //we are creating a fake screen here displaying our text in it.

            Console.WriteLine("***********************************");
            Console.WriteLine("*                                 *");
            Console.WriteLine("*                                 *");
            Console.WriteLine("*                                 *");
            Console.WriteLine("*                                 *");
            Console.WriteLine("*           "+ text + "           *");
            Console.WriteLine("*                                 *");
            Console.WriteLine("*                                 *");
            Console.WriteLine("*                                 *");
            Console.WriteLine("*                                 *");
            Console.WriteLine("*                                 *");
            Console.WriteLine("*                                 *");
            Console.WriteLine("***********************************");
            Console.WriteLine();

        }
    }
}
{% endcodeblock %}
{% codeblock lang:C# %}
using System;
namespace UniversalPrinter.Printers
{
    public class PaperPrinter
    {
        public void Print(string text)
        {
            //Actual code for sending text to a physical printer goes here

            Console.WriteLine("******* Detected printer HP LaserJet ********");
            Console.WriteLine("******* Sending Document to Printer *********");
            Console.WriteLine($"The text \"{text}\" has been sent to the printer");
            Console.WriteLine("******* End print job, retrive prinout from printer *******");
        }
    }
}
{% endcodeblock %}

4. On the Taskbar, change the build target from Debug to Release, right-click on the project and build it.
5. Right-click on the project name in Solution Explorer, select Open Folder in File Explorer. When the folder opens navigate to bin > Release > netcoreappX.X (x.x will be whatever version of dotnet core you're running). If you see a UniversalPrinter.dll file in the folder, congratulations are in order! You've created a sharable assembly.
![UniveralPrinter.dll](https://res.cloudinary.com/vnwonah/image/upload/v1573127805/up1_llvzmu.jpg)
6. Make sure to note the file path for the dll as we will be needing it later.

## Using the Printer Assembly

We will now use the assembly from another project.

1. Create a console application using Visual Studio, you can name it TestProject.
2. In Solution Explorer, right-click on Dependencies and select Add Reference. Click browse and navigate to the folder containing the dll we created. Remember the path from step 6 of the previous section?
3. Select UniversalPrinter.dll and click Add, tick the checkbox next to it if unchecked, click Okay. We have successfully added a reference to the UniversalPrinter library and can use the functionality in this new project.
4. Goto Program.cs and add the following lines of code;
{% codeblock lang:C# %}
using System;
using UniversalPrinter.Printers;
namespace TestProject
{
    class Program
    {
        static void Main(string[] args)
        {
            ScreenPrinter sc = new ScreenPrinter();
            sc.Print("Hello World!");

            PaperPrinter pc = new PaperPrinter();
            pc.Print("Hello World!");
        }
    }
}
{% endcodeblock %}

5. Run the project. We see that code from the UniversalPrinter library is executed, and text is displayed on the screen.
![Universal Printer Code Displaying text](https://res.cloudinary.com/vnwonah/image/upload/v1573128297/up2_qtijak.jpg)

We can share this Assembly (UniversalPriter.dll) with other people to reference from their applications now. They can use it but have no access to the source code.

We can also reuse the same functionality in different projects by referencing the assembly. Changes are now better handled too; any change is done only once, a new build generated and referenced.

## Conclusion

The Dotnet Framework is simply a collection of Assemblies that provides different functionalities. As an example, **Console** is a class in the **System** namespace that has a **WriteLine** method in it. This method holds functionality needed to display text to screen. We use it when we type **Console.WriteLine("some text here");**

Another article will address how we share our Assemblies with people from all over the world (hint: NuGet packages 😉)

As always, leave me a comment if you need anything clarified or have some opinion/suggestion.
