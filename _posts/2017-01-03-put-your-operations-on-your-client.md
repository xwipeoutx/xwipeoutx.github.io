---
layout: post
title: Put your operations on your client
categories:
- Programming
tags:
- c#
- typescript
- codegen
date: 2017-01-03 13:30:00 +1000
---
In the last post, [I showed how we put our server DTOs into our client code](/2016/12/put-your-server-types-on-client/), to ensure changes in our
data structures didn't silently fail. In this post, I'll show you how we protected ourselves against changing API endpoints.
<!--break-->
## Our application

As before, our application is a WebAPI project with an Angular backend in TypeScript.

We fully embraced attribute routing for this project, and put [convention tests](https://github.com/andrewabest/Conventional) in to
ensure every controller and action contains attribute routes.  Advantage: We know where to look to find URLs! Handy.

## Our goal

The goal here is to make a module that contains a way to get the URL for each operation.  For static URLs, this is just be a `const`.

For dynamic URLs, it is a function that takes in the typed parameters and returns the URL.

Examples? Sure, why not!

```
export var getOrders = `/api/orders`;
export function getOrderDetails(id: string) {
    return `/api/orders/{id}`.replace(`{id}`, encodeURIComponent(id));
}
```

Our approach was simply to loop through all these attributes and build URLs for them.

## The codes

First step is looping through all the controllers and grabbing the `RoutePrefix` and `Route` attributes, joining them together and parsing them
into something useful.

First we define a handy class to abstract away the route infos

```cs
class RouteInfo
{
    private readonly MethodInfo _method;

    public RouteInfo(MethodInfo method)
    {
        _method = method;
    }

    public string Name => (_method.DeclaringType.Name.Substring(0, 1).ToLower() + _method.DeclaringType.Name.Substring(1)).Replace("Controller", "");

    public string FullRouteTemplate => $"/{Prefix}/{Route}";

    private string Prefix => AttributeOnMethodOrType<RoutePrefixAttribute>().Prefix;
    private string Route => AttributeOnMethodOrType<RouteAttribute>().Template;

    private T AttributeOnMethodOrType<T>() where T : Attribute
    {
        return _method.GetCustomAttribute<T>()
                ?? _method.DeclaringType.GetCustomAttribute<T>();
    }
}
```

Then we make route infos from the action methods.  We do a poor man's distinct by on the full template here, perhaps overly and erroneously
defensive, but there you have it.

```cs
var actionMethods = typeof(FooController).Assembly
    .GetLoadableTypes()
    .Where(t => t.IsSubclassOf(typeof(ApiController)))
    .SelectMany(t => t.GetMethods(BindingFlags.Public | BindingFlags.Instance | BindingFlags.InvokeMethod | BindingFlags.DeclaredOnly))
    .ToList();

var routes = actionMethods
    .Select(m => new RouteInfo(m))
    .GroupBy(r => r.FullRouteTemplate)
    .Select(g => g.First())
    .ToList();
```

Then use some rad regex and munging to make typescript from these routes. Here we go...

```cs
{% raw %}
private static readonly Regex ParamsRegex = new Regex(@"\{(?<name>[^?\}:]+):?(?<constraint>[^?\}]+)?(?<isOptional>[^\}]+)?\}");
private static readonly Regex ParamsReplaceRegex = new Regex("\\:[^}]+");

private static string BuildRouteLine(RouteInfo route)
{
    if (!route.FullRouteTemplate.Contains("{"))
        return $"export var {route.Name} = `{route.FullRouteTemplate}`;";

    var parameters = ParamsRegex.Matches(route.FullRouteTemplate)
        .Cast<Match>()
        .Select(m => new
        {
            Name = m.Groups["name"].Value,
            Constraint = m.Groups["constraint"].Value,
            IsOptional = m.Groups["isOptional"].Success
        })
        .ToList();

    var parameterNamesAndTypes = string.Join(", ", parameters.Select(p => $"{p.Name}: {ConvertRouteConstraintToTsType(p.Constraint)}"));

    var routeWithoutTypeNames = ParamsReplaceRegex.Replace(route.FullRouteTemplate, string.Empty);
    
    var functionContents = parameters.Aggregate($"return `{routeWithoutTypeNames}`", (current, parameter) => ReplaceParameter(current, parameter.Name, parameter.IsOptional)) + ";";

    return $"export function {route.Name}({parameterNamesAndTypes}) {{{Environment.NewLine}    { functionContents }{Environment.NewLine}}}";
}

private static string ConvertRouteConstraintToTsType(string constraint)
{
    switch (constraint)
    {
        case "alpha":
        case "guid":
        case "datetime":
            return "string";
        case "decimal":
        case "double":
        case "float":
        case "int":
        case "long":
            return "number";
        case "bool":
            return "Boolean";
        default: // No type mappings for constraints like length, max, maxlength, min, minlength, range, & regex
            return "any";
    }
}
{% endraw %}
```

Let's go through it bit by bit:

```cs
if (!route.FullRouteTemplate.Contains("{"))
    return $"export const {route.Name} = `{route.FullRouteTemplate}`;";
```

If there are no curlies, then there are no parameters - just output the template as-is - easy as!

```cs
var parameters = ParamsRegex.Matches(route.FullRouteTemplate)
    .Cast<Match>()
    .Select(m => new
    {
        Name = m.Groups["name"].Value,
        Constraint = m.Groups["constraint"].Value,
        IsOptional = m.Groups["isOptional"].Success
    })
    .ToList();
```

Here we make anonymous types for all the constraints using our regex.  Of course, the regex is the most crazy part of this whole thing,
but what it essentially does it grab `id`, `guid` and `optional` from something like `{id:guid?}`.  It will match multiple in something
like `/api/orders/{status:string}/{top:int?}` (that's a terrible URL, don't use it in your project).

```cs
var parameterNamesAndTypes = string.Join(", ", parameters.Select(p => $"{p.Name}: {ConvertRouteConstraintToTsType(p.Constraint)}"));
```

This chunk makes a string that looks like `status: string, top?: number` - these will be the parameters of the generated method.  
Note we convert the route types to TS friendly types using a good ol' `switch` statement, falling back to `any` when they do not match.

```cs
var routeWithoutTypeNames = ParamsReplaceRegex.Replace(route.FullRouteTemplate, string.Empty);

var functionContents = parameters.Aggregate($"return `{routeWithoutTypeNames}`", (current, parameter) => ReplaceParameter(current, parameter.Name, parameter.IsOptional)) + ";";
```

```cs
{% raw %}
private static string ReplaceParameter(string current, string paramName, bool isOptional)
{
    return isOptional
        ? $"{current}.replace(`/{{{paramName}}}`, {paramName} === undefined ? `` : `/${{encodeURIComponent({paramName})}}`)"
        : $"{current}.replace(`{{{paramName}}}`, encodeURIComponent({paramName}))";
}
{% endraw %}
```

This part changes our route `/api/orders/{status:string}/{top:int?}` to  `/api/orders/{status}/{top}`, which makes it suitable for ts substitution.

We then just make function contents do a fairly naive string replace of those parameters.  It ends up looking like:

```
    `/api/orders/{status}/{top}`.replace(`{status}`, encodeURIComponent(status)).replace(`{top}`, top === undefined ? `` : encodeURIComponent(top));
```

We rarely used optional parameters, so it was pretty naive implementation.

```cs
{% raw %}
return $"export function {route.Name}({parameterNamesAndTypes}) {{{Environment.NewLine}    { functionContents }{Environment.NewLine}}}";
{% endraw %}
```

This bit glues all the others bits together, and we're done making the function contents.

:boom:

## Writing it out.

We have route infos, and a way of making TS code for those routes.  Now let's jam it in a file:

```cs
var builder = new StringBuilder();

foreach (var route in routes)
{
    builder.AppendLine(BuildRouteLine(route));
}

File.WriteAllText("urls.ts", builder.ToString());
```

Done! We win at putting urls on clients!

## Using it

We wrapped all our HTTP calls into service classes, which looked thus:

```ts
import * as urls from "../urls";

export class OrderService {
    static $inject = ["$http"];
    constructor(private $http: ng.IHttpService) {

    }

    orders() {
        return this.$http.get<Api.OrderSummary[]>(urls.getOrders)
            .then(result => result.data);
    }

    getOrder(id: string) {
        return this.$http.get<Api.OrderDetails>(urls.getOrderDetails(id))
            .then(result => result.data);
    }

    saveOrder(order: Api.OrderDetails) : ng.IPromise<void> {
        return this.$http.post(urls.saveOrder(order.id), params)
            .then(() => { })); // Makes return type void
    }
}
```

## Going further

This could be automated too - by looking at the HTTP method and body-valued things, but we had enough custom logic in our services
that this wasn't worthwhile. For instance, some HTTP calls would show/hide loading indicators, or handle errors differently. We felt this
was a happy medium with a relatively simple implementation.

Automating this bit would have taken a fair chunk of time - we would have had to marry up the route infos with the controller actions/types - and while
this is fairly straight forward for return types, parameter types get pretty hairy with `[FromBody]` and `[FromUri]` attributes, optional arguments, 
primitive vs class identification and the different handling for POST and GET.  

And even then we'd still have to wrap _that_ for reasons mentioned above. In the end, compile time URL validation was good enough, 
and saved us tonnes of time. You should do it too!