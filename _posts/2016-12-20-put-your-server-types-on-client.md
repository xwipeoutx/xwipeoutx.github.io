---
layout: post
title: Put your server types on your client
categories:
- Programming
tags:
- c#
- typescript
- codegen
date: 2016-12-20 14:00:00 +1000
---
A recent gig I was involved in relied fairly heavily on code generation in order to make our client/server communications type safe.  

We were using [TypeScript](https://www.typescriptlang.org) so a lot of our safety could be guaranteed at compile time - as long as the types were on the client.
Since we're fallibe, and computers like doing things repetitively, we used some codegen to do thish for us.

Our goals?

- Generate DTOs for all models (and dependants) going to/from API controllers
- For all enums in the models, create a way to get names, values and descriptions
- Avoid TypeScript's `any` keyword, which breaks this whole approach

I will be covering URL and client generation in another post - this is about the _types_ not the _operations_.

## Generating Defintions: TypeLite

[TypeLite](TypeLite) did the bulk of the heavy lifting for us - it had appropriate extensibility points, making it easy
to customise things like camel casing and type names.

We found the easiest thing to do was to add a command like project to our solution,
and a wrapper powershell script to compile and execute it.  This project could then
just point to the assembly we wanted types for and away we go!

Here's what our generator looked like:

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
            .Where(t => !t.IsPrimitive && t != typeof(string) && t != typeof(object))
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
* Spaces, not tabs.

### Aside: Why not NSwag?

This isn't the only option - [NSwag](https://github.com/NSwag/NSwag) can do a similar thing in a few lines of code:

```cs
var controllers = typeof(FooController).Assembly.GetExportedTypes()
    .Where(t => t.IsSubclassOf(typeof(ApiController)));

var document = new WebApiToSwaggerGenerator(new WebApiToSwaggerGeneratorSettings())
    .GenerateForControllers(controllers);

var code = new SwaggerToTypeScriptClientGenerator(document, new SwaggerToTypeScriptClientGeneratorSettings())
    .GenerateFile();

File.WriteAllText("api.ts", code);
```

Unfortunately it didn't fill the bill for us for a few reasons:

* No customization of client side type names - specifically we wanted classes
to be `PascalCase` and properties / parameters to be `camelCase`, which didn't seem possible
* Generated rather verbose classes for contracts instead of simple interfaces that would not be in the output.
  * These classes were also very `any` friendly, which made it too easy to dodge
* We were going to write our own abstraction over the HTTP services anyway, for more simply customizable loading bars and things
* No insight into original C# types.  The `enum` support was pretty important to us

## Generating Enums: EnumGenie

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
grab descriptions and use the enums as we want!  

One cool little feature of TypeScript is that these are treated as equivalent, and can be assigned to each other:
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

## Conclusion

There's a fair chunk of code here - most of it reflecting over WebAPI and fiddling
with formatting.  The result is a tonne of generated goodies for use on the client.

Stay tuned for the next post, where we'll look at getting the operations themselves
down to the client.