---
layout: post
title: Chaining Expressions in C#
categories:
- Programming
tags:
- c#
- expressions
- linq
date: 2016-06-06 22:00:00 +1000
---
A recent gig I was involved in relied fairly heavily on code generation in order to make our client/server communications safe.  

We were using [TypeScript](https://www.typescriptlang.org) so a lot of our safety could be guaranteed at compile time - as long as the types were on the client.

Our goals?

- Generate DTOs for all models (and dependants) going to/from API controllers
- For all enums in the models, create a way to get names, values and descriptions
- URLs for all API endpoints.

## Using TypeLite

[TypeLite](TypeLite) did the bulk of the heavy lifting for us - it had appropriate extensibility points, making it easy
to customise things like camel casing and type names.

We found the easiest thing to do was to add a command like project to our solution,
and a wrapper powershell script to compile and execute it.  This project could then
just points to the assembly we wanted types for and away we go!

Our configuration was like this:

```cs
public class Program
{
    public static void Main()
    {
        var apiTypes = typeof(FooController).Assembly.GetExportedTypes()
            .Where(t => t.IsSubclassOf(typeof(ApiController)))
            .SelectMany(type => type.GetMethods(BindingFlags.Public | BindingFlags.Instance | BindingFlags.InvokeMethod))
            .SelectMany(ParameterAndReturnTypes)
            .SelectMany(Unwrap)
            .Where(t => !t.IsPrimitive && t != typeof(string))
            .Where(t => !t.Namespace.StartsWith("System.")) // Customize this bit to suit your app
            .Distinct()
            .OrderBy(t => t.FullName) // Makes output better for diff
            .ToList();

        var typeScriptFluent = TypeScript.Definitions()
            .WithIndentation("    ")
            .WithModuleNameFormatter(tsModule => "Api")
            .WithConvertor<DateTimeOffset>(obj => "string") // You may need to add more of these
            .WithConvertor<Guid>(obj => "string")
            .WithMemberFormatter(mf =>
            {
                // Hack to ignore statics - limitation of TypeLite
                if ((mf.MemberInfo as PropertyInfo)?.GetGetMethod().IsStatic ?? false)
                    return $"// Ignore static: {mf.Name}";

                // Hack to mark nullables as such- limitation of TypeLite
                var suffix = ((mf.MemberInfo as PropertyInfo)?.PropertyType.IsNullable() ?? false) ? "?" : "";
                return $"{char.ToLower(mf.Name[0])}{mf.Name.Substring(1)}{suffix}";
            })
            .AsConstEnums(false);

        foreach (var type in apiTypes)
        {
            typeScriptFluent = typeScriptFluent.For(type);
        }

        var tsModel = typeScriptFluent.ModelBuilder.Build();

        File.WriteAllText("api.d.ts", typeScriptFluent.Generate());
    }

    private static IEnumerable<Type> ParameterAndReturnTypes(MethodInfo method)
    {
        return method.GetParameters().Select(p => p.ParameterType)
            .Concat(new[] { method.ReturnType })
            .Distinct();
    }

    private static IEnumerable<Type> Unwrap(Type type)
    {
        return type.IsGenericType ? UnwrapGeneric(type)
            : type.IsArray ? UnwrapArray(type)
                : new[] { type };
    }

    private static IEnumerable<Type> UnwrapArray(Type type)
    {
        return Unwrap(type.GetElementType());
    }

    private static IEnumerable<Type> UnwrapGeneric(Type type)
    {
        return type.GenericTypeArguments.SelectMany(Unwrap);
    }
}
```

A few tidbits with this config:

* We do our own assembly scanning - TypeLite didn't do it the way we wanted, so we scanned it ourselves
* `System.` namespaces classes are ignored - you can whitelist this bit to your own assembly if that suits better
* Generic types and arrays need to be unwrapped recursively to ensure they get included explicitly
* Certain types need to be overridden because `Date` and `Guid` are not things in JSON land
* Order by fullname makes diffs nice!
* Spaces, not tabs

## Generating enum bits

This worked fine for our model, and we got enum declarations as a result, but code
like `let status = OrderStatus.InProgress` cannot be used - the generated file isn't compiled
to anything.

So we wrote some code to generate all the useful enum bits.  I've since made a library out of it, [EnumGenie](https://github.com/xwipeoutx/EnumGenie).

A bit of code added to the above

```cs
new EnumGenie.EnumGenie()
    .SourceFrom.List(tsModel.Enums.Select(e => e.Type))
    .WriteTo.File("enums.ts", cfg => cfg.TypeScript())
    .Write();
```

and whamo! Suddenly we have a magical file letting us loop through enum values,
grab descriptions and use the enums as we want!  One cool little feature of
TypeScript is that these are treated as equivalent:
```ts
// Api.d.ts
declare export enum Status {
    InProgress,
    Complete
}

// Enums.ts
export enum Status {
    InProgress,
    Complete
}

```

So an enum generated from EnumGenie can be assigned to the API enum wih no issue!

## URLs

Our URL solution was a little rudimentary.  We had a [convention](https://github.com/andrewabest/Conventional) of one-public-method-per-controller,
which meant that mappings between controllers and endpoints were 1-1.

Aside: I wouldn't do this again, as it meant we couldn't have `POST` and `GET` on the same URL, but it made URL generation easy.

Generating the URLs was then a matter of reflecting over the controllers, grabbing our route attributes (mandatory via a convention test) and jamming them together.

We put in a couple of smarts to support route parameters to, but the resulting code was something like:

```cs
```