---
title: Casting Between Custom Types in C#
toc: true
category: C#
date: 2019-11-19 12:29:00
tags:
    - dotnet core
    - C#
    - architecture
---

This is a follow up from the article on [Casting Custom Types To C# Built-In Types.](~/../casting-custom-types-to-c-sharp-built-in-types.md)

In the previous article we looked at how we would take a custom object like a Class or a Struct and cast it to a C# built-in like a bool. In this article we will look at how to cast from one custom type to another.

What we will be exploring here is much like what [AutoMapper](http://automapper.org/) does.

<!--More-->

## Implementing the ToType() method on IConvertible

For this part we will define two custom classes, intialize one, then try casting it to the second custom type. See code below

{% codeblock lang:C# %}

//the class below implements the IConvertble interface
class PersonDTO : IConvertible
{
    public string Name { get; set; }

    public object ToType(Type conversionType, IFormatProvider provider)
    {
        if (!(conversionType is object))
            throw new ArgumentNullException("Argument conversionType cannot be null");
        if(conversionType == typeof(PersonEntity))
        {
            var nameArray = Name.Split(" ");
            return new PersonEntity { FirstName = nameArray[0], LastName = nameArray[1] };
        }
        throw new ArgumentException($"cannot cast PersonDTO to {conversionType.Name}", conversionType.Name);
    }
}

public class PersonEntity
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
}
{% endcodeblock %}

The `PersonDTO` class implements the `IConvertible` interface (I am showing only relevant method here). It takes takes a Type argument and returns a new object of that type; in this case a new `PersonEntity.` See usage below:

{% codeblock lang:C# %}

//create a new PersonDTO
PersonDTO personDTO = new PersonDTO { Name = "Vincent Nwonah" };

//cast the PersonDTO to a PersonEntity
PersonEntity personEntity = Convert.ChangeType(personDTO, typeof(PersonEntity));

{% endcodeblock %}

when executed, the `ToType()` method on `PersonDTO` is called and returns a new `PersonEntity` with the first and last name set.

## Conclusion

We can add multiple if else statements in the `ToType()` method to convert the `PersonDTO` class to other types if need be. This gives us a way to convert between types in cases where we do not want to, or cannot bring in libraries like  [AutoMapper](http://automapper.org/).
