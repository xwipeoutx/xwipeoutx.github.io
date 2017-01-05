---
layout: post
title: Seedy Fake Data
categories:
- Programming
tags:
- c#
- seed data
- test
date: 2017-01-05 10:45:00 +1000
---
With fake user authenication done, we had everything we needed to generate fake data.

The seed data was generated on every deployment to our dev and demo environments - which gave us nice, clean, predictable demos,
and our dev server was never a terrible mess of temporary data (yeah, you know what I'm talking about).

<!--break-->

## Our approach

Our server side was running WebAPI and we had strong types for all our commands, which made it easy to use the same types for communicating 
with it.  We also took CQS fairly seriously at the API level, which greatly simplified everything - we generally didn't have to worry about
parsing return values at all.

We had already written some [subcutaneous tests](http://martinfowler.com/bliki/SubcutaneousTest.html) for our API, and they all ran on the
[OWIN test server](https://msdn.microsoft.com/en-us/library/microsoft.owin.testing.testserver(v=vs.113).aspx).  We'd already seperated out
separate clients (grouped loosely by aggregate type) that wrapped the HTTP client, meaning we already had a handy way to call API methods from C#.

So creating seed data was just a matter of defining data structures that mapped to commands to throw at our API.

## What data do I put in?

Seed data works best if it is up to date - so we made a point of ensuring that **every time a story was completed, that there
was seed data for that story**.  If nothing else, it provided assurances of the happy-path usage of our API.  But more usefully,
when it came time for sprint review, we knew the data was there, and was identical to the data we'd been using on dev the whole time.

Most of the data we generated was hard coded - it was relatively important to our client to have realistic and professional looking data.
It certainly had its fair share of `Lorem Ipsum` still, but we didn't use [Bogus](https://github.com/bchavez/Bogus) for this part.  Depending
on your domain, using bogus could be a very sensible thing to do, though.

## Why at the API level?

A common way is to seed data directly into the DB, but we found this was pretty limiting.  The main value of using the API is that our
whole environment is ready - specifically, our service bus messages are created, fired off and handled.  Our whole system is seeded, not just
the database - this is very valuable.

Apart from that, we had assurance of our happy-path, and early warning of any slowdowns.  You can generate pretty large amounts of data
in the sample - and it's relatively fast too because of the high level of parallelism. If your seed data takes minutes to generate a few
hundred entities, then you'll get frustrated enough to fix your performance problems.

That said, there was one part where we twiddle it in the database - some dates were calculated based on the current server time,
and our feature was about having warnings for old data.  This approach doesn't prevent hacking in a DB update like that.

## How we did it

There's a lot of code here - but I thought it worth sharing, because the approach was very successful and flexible. I'll 
certainly be copy/pasting from this post in a future project.

To start with, we do some initial plumbing to make the API as simple to consume as we can,
with appropriate error reporting.  Then we make some declarative data structures to represent the seed data, and map them API calls.

### HTTP Clients

`System.Net.Http.HttpClient` is rather low level, working with `HttpContent` and requiring manual insertion of things like bearer tokens and
XSRF protection - so we wrote some extension methods which proved quite helpful:

```cs
public static class ApiHttpClientExtensions
{
    public static async Task<HttpResponseMessage> GetAsync(this HttpClient httpClient, string url, IUser user)
    {
        var request = await CreateRequestAsync(HttpMethod.Get, url, user);
        return await httpClient.SendAsync(request);
    }

    public static async Task<HttpResponseMessage> PostAsync<T>(this HttpClient httpClient, string url, IUser user, T command)
    {
        var request = await CreateRequestAsync(HttpMethod.Post, url, user);
        request.Content = new StringContent(JsonConvert.SerializeObject(command), Encoding.UTF8, "application/json");
        return await httpClient.SendAsync(request);
    }

    public static async Task<HttpResponseMessage> PostStreamAsync(this HttpClient httpClient, string url, IUser user, FileReference fileReference)
    {
        var request = await CreateRequestAsync(HttpMethod.Post, url, user);
        request.Content = new StreamContent(fileReference.Stream);
        request.Content.Headers.ContentType = new MediaTypeHeaderValue(fileReference.MimeType);
        return await httpClient.SendAsync(request);
    }
    
    // Continue for PATCH, DELETE and so on

    private static async Task<HttpRequestMessage> CreateRequestAsync(HttpMethod method, string url, IUser user)
    {
        var request = new HttpRequestMessage(method, url);
        if (user != null)
        {
            request.Headers.Authorization = await CreatAuthorizationHeaderForUser(user);
            request.Headers.Add(XsrfProtection.XsrfTokenHeaderName, XsrfProtection.GenerateXsrfCode(user.Id, ConfigurationManager.AppSettings["XsrfProtectionServerSecret"]));
        }
        return request;
    }

    private static async Task<AuthenticationHeaderValue> CreatAuthorizationHeaderForUser(IUser user)
    {
        return new AuthenticationHeaderValue("Bearer", await TestBearerTokenGenerator.CreateBearerToken(user));
    }
}
```

These extension methods do a lot of our work for making our server calls for us, and let us work with the raw message.  The 
user token is also thrown in here - this bit will depend on your auth mechanism, so do with it as you will.

### Checking for errors

Our subcutaneous tests did a lot of checking for errors in responses and reporting them.  Here's the extension methods we used for that.

```cs
public static class ResponseExtensions
{
    public static async Task<T> ParseSuccessResponseAsync<T>(this Task<HttpResponseMessage> response)
    {
        return await ParseSuccessResponseAsync<T>(await response);
    }

    public static async Task<T> ParseSuccessResponseAsync<T>(this HttpResponseMessage response)
    {
        if (!response.IsSuccessStatusCode)
            throw new ArgumentException($"Expected a success status code, but got {response.StatusCode} for {response.RequestMessage.RequestUri} ", nameof(response));

        var asString = await response.Content.ReadAsStringAsync();
        return TryDeserialize<T>(asString);
    }

    public static async Task<HttpResponseMessage> AndCheckForErrors(this Task<HttpResponseMessage> response)
    {
        return await AndCheckForErrors(await response);
    }

    public static async Task<HttpResponseMessage> AndCheckForErrors(this HttpResponseMessage response)
    {
        if (response.IsSuccessStatusCode != true)
        {
            var asString = await response.Content.ReadAsStringAsync();
            var errorResponse = TryDeserialize<ErrorResponse>(asString);

            var message = errorResponse?.Message ?? asString;

            throw new Exception($"Response for {response.RequestMessage.RequestUri} failed with HTTP {response.StatusCode}: \"{message}\"");
        }

        return response;
    }

    private static T TryDeserialize<T>(string text)
    {
        try
        {
            return JsonConvert.DeserializeObject<T>(text, Serializers.ApiSettings);
        }
        catch (Exception)
        {
            return default(T);
        }
    }
}
```

With all this put together, we could make API calls as specific users using our own command objects and ensure that everything was successful.

We could also pull data from our API and parse it very simply.

### Domain-specific clients

We used these extension methods to create aggregate-specific HTTP clients:

```cs
public class ProductHttpClient
{
    private readonly HttpClient _httpClient;
    private readonly IUser _user;

    public ProductHttpClient(HttpClient httpClient, IUser user)
    {
        _httpClient = httpClient;
        _user = user;
    }

    public ProductHttpClient AsUser(IUser user)
    {
        return new ProductHttpClient(_httpClient, user);
    }

    public Task<HttpResponseMessage> Create(Guid productId, string name)
    {
        var command = new CreateProductCommand
        {
            Id = productId,
            Name = name
        };
        return _httpClient.PostAsync("/api/products/create", _user, command);
    }

    public Task<HttpResponseMessage> ChangePrice(Guid productId, decimal price)
    {
        var command = new ChangeProductPriceCommand()
        {
            Id = productId,
            Price = price
        };
        return _httpClient.PostAsync("/api/products/change-price", _user, command);
    }
    
    public Task<HttpResponseMessage> Activate(Guid productId)
    {
        var command = new ActivateProductCommand()
        {
            Id = productId
        };
        return _httpClient.PostAsync("/api/products/activate", _user, command);
    }
    
    public Task<ProductDetails> Details(Guid productId)
    {
        return _httpClient.GetAsync($"/api/products/{productId:D}", _user).ParseSuccessResponseAsync<ProductDetails>();
    }
}
```

Note we still return the raw HTTP requests for commands.  This is a hangover from our 
subcutaneous tests, where we test that an error status code is returned appropriately.

### Seeding it all

All we need now is a definition of our destination data, and a way to get from nothing to that definition.

```cs
public class ProductDefinition 
{
    public Guid Id { get; set; }
    public string Name { get; set; }
    public decimal? Price { get; set; }
    public bool IsActive { get; set; }
}

public class ProductBuilder
{
    private readonly ProductHttpClient _httpClient;

    public ProductBuilder(ProductHttpClient httpClient) 
    {
        _httpClient = httpClient;
    }

    public async Task Build(ProductDefinition product) 
    {
        await _httpClient.Create(product.Id, product.Name).AndCheckForErrors();

        if (product.Price != null)
            await _httpClient.ChangePrice(product.Id, product.Price.Value).AndCheckForErrors();

        if (product.IsActive)
            await _httpClient.Activate(product.Id).AndCheckForErrors();
    }
}

public class ProductSeedData
{
    private readonly ProductBuilder _productBuilder;
    
    public ProductSeedData(HttpClient httpClient)
    {
        var productHttpClient = new ProductHttpClient(httpClient, TestUsers.Staff.FuzzyMcStickpants);
        _productBuilder = new ProductBuilder(productHttpClient);
    }

    public async Task SeedAsync()
    {
        await Task.WhenAll(
            Products.Select(_productBuilder.Build)
        );
    }

    public static readonly IReadOnlyList<ProductDefinition> Products = new[] 
    {
        new ProductDefinition {
            Id = Guid.NewGuid(),
            Name = "Bike",
            Price = 800,
            IsActive = true
        },
        new ProductDefinition {
            Id = Guid.NewGuid(),
            Name = "Trampoline",
            Price = 120,
            IsActive = true
        },
        new ProductDefinition {
            Id = Guid.NewGuid(),
            Name = "Helmet"
        }
    };

    public static ProductDefinition Bike => Products[0];
    public static ProductDefinition Trampoline => Products[1];
}

// NOTE: Order definition/builders omitted for brevity

public class OrderSeedData
{
    private readonly OrderBuilder _orderBuilder;
    
    public OrderSeedData(HttpClient httpClient)
    {
        var orderHttpClient = new OrderHttpClient(httpClient, TestUsers.Customers.BaloneyMaloney);
        _orderBuilder = new OrderBuilder(orderHttpClient);
    }

    public async Task SeedAsync()
    {
        await Task.WhenAll(
            Orders.Select(_orderBuilder.Build)
        );
    }

    public static readonly IReadOnlyCollection<OrderDefinition> Orders = new[] 
    {
        new OrderDefinition {
            Id = Guid.NewGuid(),
            ProductIds = { ProductSeedData.Bike.Id, ProductSeedData.Trampoline.Id }
        }
    };
}
```

Note we kept all our definitions `static` so we could reference other ids that were created during the seed process. Our ids were 
_never_ hardcoded, to prevent issues when running multiple times.  Well, I say _never_, but that's a lie - people did, 
and it did cause issues.

Finally, we threw it in a console program which ran after our deployment.

```cs
var baseUrl = ConfigurationManager.AppSettings["BaseUrl"];

var httpClient = HttpClientFactory.Create(new HttpClientHandler());
httpClient.BaseAddress = new Uri(baseUrl);

var productSeedData = new ProjectSeedData(httpClient);
var orderSeedData = new OrderSeedData(httpClient);

await productSeedData.SeedAsync();
await orderSeedData.SeedAsync();
```

And we're one octostep away from done, but that's an exercise for the reader.

## Some gotchas

Of course there were some issues!

There were days when we were all checking in frequently - which meant lots of deployments to dev. Which meant our data kept getting
blown away.  Which meant we couldn't smoke-test our features in the dev environment easily.  If this happened more frequently, we could
have mitigated it by making the seed data happen on a nightly deployment, instead of every deployment or something.  But it wasn't a big issue.

THe network speed was a limitation - our project had image upload, and we wanted good quality images for the demo, so they were large.  We 
were in a government building so (sigh) we had very limited network speed.  One of the biggest speed-ups we had in our seed data was
shrinking our sample PNG images down.

It's also minldy annoying when you forget to nuke your storage or run your web server before running locally.  Ensure you've got all your
scripts to reset the world and everything around for this.

Happy seeding!