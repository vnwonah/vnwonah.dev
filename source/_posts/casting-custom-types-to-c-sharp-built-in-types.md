---
title: Casting Custom Types To C# Built-In Types
toc: true
category: C#
date: 2019-11-15 14:36:52
tags:
    - dotnet core
    - C#
    - architecture
---

## Quick Quiz

What does the code below do?

{% codeblock lang:C# %}
CustomObject c1 = new CustomObject()
bool variableName = Convert.ToBoolean(c1);

//1. variableName is false
//2. runtime exception
//3. It depends
{% endcodeblock %}

If you're not sure what the answer is, read on - all will be clear.

<!--More-->

## The Convert.ToBoolean(object value); method

Let us Investigate how Conversions with the Convert.ToBoolean method works.

C# has a Class - [Convert.cs](https://github.com/dotnet/corefx/blob/master/src/Common/src/CoreLib/System/Convert.cs) that defines several methods. Some of these methods are:

{% codeblock lang:C# %}
public static bool ToBoolean(object value) {}
public static int ToInt32(object value) {}
{% endcodeblock %}
...

and many others, with overloads for each. Because these methods are static, we can just use them without Initiaizing an object as

    Convert.ToBoolean(objectInstance);

How this "conversion" works internally is pretty straight-forward. The static `Convert.ToBoolean` method on the `Convert` Class depends on the object to be converted to tell it how to do the Conversion. This interaction requires that any object that wants to be converted using these static methods must implement the [IConvertible](https://github.com/dotnet/corefx/blob/master/src/Common/src/CoreLib/System/IConvertible.cs) Interface. Let's look at some code.

## Implemeting IConvertible

Define a class point with two properties X and Y

{% codeblock lang:C# %}
class Point {
    public int X {get;set;}
    public int Y {get;set;}
}
// Create a new instance of Point
Point p0 = new Point();
//p0.X and p0.Y are currently at 0,0
//showing our point hasn't moved.
{% endcodeblock %}

Suppose we want to use a bool value to show that our point has moved, how would we handle this? There are several ways but for the purpose of this article let us convert our point to a bool, and check if its a true or false value; True if either X or Y is no longer 0 (point has changed) and false otherwise. We do this by calling

`bool hasPointMoved = Convert.ToBoolean(p0);`

if we execute this at this point we get a runtime exception `System.InvalidCastException: 'Unable to cast object of type 'Point' to type 'System.IConvertible'.'` This is because our point class doesn't implement the IConvertible Interface. We implement it below and use Visual Studio to generate the necessary methods we need to implement. (I am showing only ToBoolean here).

{% codeblock lang:C# %}
class Point {

    public int X;
    public int Y;

    //the Convert Class will call into this
    //ToBoolean for conversion to happen
    public bool ToBoolean(IFormatProvider provider)
    {
        return (X != 0 && Y != 0);
    }

    //the method simply returns true if either of X and Y has changed, and false otherwise
}
{% endcodeblock %}

Run the code now and you see our conversion returns `false`, alter either X or Y to something other than 0 and we get a `true`.

## Conclusion

Circling back to answer the question at the top of the article; the correct answer is "It depends", on if our CustomClass has an appropriate implementation of the IConvertible Interface.

C# built-in types like string, int, bool, DateTime etc follow this same pattern to Implement the IConvertible Interface and specify what value is retured when they are converted. As always, leave a comment, suggestion, or improvement if you have any in the comments.
