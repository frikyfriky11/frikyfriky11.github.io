+++
date = 2023-06-04T22:46:38+02:00
title = "AutoMapper between classes and records"
description = "Sometimes you need to map classes and records that have different parameters, and AutoMapper can give a confusing error."
authors = ["Stefano Previato"]
tags = ["AutoMapper", "C#", "mapping"]
categories = ["C#", "Programming"]
+++

Mapping objects between different classes or data structures is a common task in software development. It allows us to transform data from one representation to another. [AutoMapper](https://automapper.org/) is a powerful library that simplifies this process by automatically mapping properties based on naming conventions. However, when working with [records](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/record), there are some specific considerations to keep in mind to ensure seamless mapping. In this blog post, I will share an issue I encountered while using AutoMapper v12.0.1 to map properties between a class and a record, along with the solution I found to overcome it.

## Understanding the problem

Let's say we have a class called `SourceClass`:

```csharp
public class SourceClass {
    public Guid Id { get; set; }
    public string? Name { get; set; }

}
```

and a record called `DestinationRecord`:

```csharp
public record DestinationRecord(Guid Id, string? Name, byte[] Content);
```

If you try to map two instances of these two objects (for example using something like `_mapper.Map(sourceClass, destinationRecord)`) you may receive the following error at run-time:

> System.ArgumentException: Type 'DestinationRecord' does not have a default constructor (Parameter 'type')

This error is absolutely confusing if you're used to the usual errors that AutoMapper returns (such as `Missing map` or other similar errors) and will make you lose incredible amounts of time trying to debug the issue.

This occurs specifically because of how records and classes work in C#: records have a default constructor that is not empty even when you do not declare one, because the signature of the record itself is a constructor, assuming the record is declared in the usual way.

If you are curious like me, this is what our example record looks like when lowered to "normal" C# (with all the `CompilerGenerated` and other attributes stripped away for brevity as well as other auto-generated methods, if you want to see them use something like [sharplab.io](https://sharplab.io/)):

```csharp
public class DestinationRecord : IEquatable<DestinationRecord>
{
    private readonly Guid <Id>k__BackingField;

    private readonly string <Name>k__BackingField;

    private readonly byte[] <Content>k__BackingField;

    public Guid Id
    {
        get { return <Id>k__BackingField; }
        init { <Id>k__BackingField = value; }
    }

    public string Name
    {
        get { return <Name>k__BackingField; }
        init { <Name>k__BackingField = value; }
    }

    public byte[] Content
    {
        get { return <Content>k__BackingField; }
        init { <Content>k__BackingField = value; }
    }

    public DestinationRecord(Guid Id, string Name, byte[] Content)
    {
        <Id>k__BackingField = Id;
        <Name>k__BackingField = Name;
        <Content>k__BackingField = Content;
        base..ctor();
    }

    public virtual DestinationRecord <Clone>$()
    {
        return new DestinationRecord(this);
    }

    protected DestinationRecord(DestinationRecord original)
    {
        <Id>k__BackingField = original.<Id>k__BackingField;
        <Name>k__BackingField = original.<Name>k__BackingField;
        <Content>k__BackingField = original.<Content>k__BackingField;
    }

    // other methods omitted for brevity
}
```

As you can see there are actually two constructors that are generated automatically from our code, one is the default public constructor with the signature `public DestinationRecord(Guid Id, string Name, byte[] Content)` and the other one is a protected constructor `protected DestinationRecord(DestinationRecord original)`. The first one is what you would use if you needed to create an instance of this record by specifying the value of each property, and the second one is used by the `Clone` method.

Since records declare both `{ get; init; }` properties (because records are immutable by default) and a public constructor, AutoMapper cannot successfully create an instance of the `DestinationRecord` object because in our example case the constructor contains three properties, all of them required because none is declared as optional with ` = null` (and no, marking them as nullable doesn't change anything), but the source object (our `SourceClass`) only has two of them that can be mapped by name. When AutoMapper tries to create an instance of the object, it fails miserably telling us that it didn't find a suitable constructor that has the parameters it expected to have.

## The solution: ignoring a property in the record constructor

After some investigation and searching for solutions, I discovered a workaround to make AutoMapper successfully map between the class and the record. By instructing AutoMapper to ignore a specific property in the record constructor, we can overcome this issue. Here's how I implemented it:

```csharp
CreateMap<SourceClass, DestinationRecord>()
  .ForCtorParam(nameof(DestinationRecord.Content), options => options.MapFrom(source => (byte[])null!));
```

In this code snippet, I used the `CreateMap` method provided by AutoMapper to define the mapping configuration between the source (`SourceClass`) and destination (`DestinationRecord`) types. To ignore the `Content` property in the record constructor, I utilized the `ForCtorParam` method. By specifying the property name (`nameof(DestinationRecord.Content)`) and providing a mapping expression, I was able to tell AutoMapper to exclude this property during the mapping process. In this case, I assigned a null value to the `Content` property using the `MapFrom` method.

## Conclusion

Mapping between different classes and data structures is an essential part of software development. When using AutoMapper to map between classes and records, it's important to consider specific challenges that may arise. In this blog post, I shared an issue I encountered while mapping properties between a class and a record and provided a solution to overcome it. By instructing AutoMapper to ignore a property in the record constructor, we can successfully map between the two types. It's crucial to be aware of such scenarios and leverage the features and configurations offered by AutoMapper to ensure smooth and efficient data transformation in our applications.
