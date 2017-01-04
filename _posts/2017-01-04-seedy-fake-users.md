---
layout: post
title: Seedy Fake Users
categories:
- Programming
tags:
- c#
- seed data
- test
date: 2017-01-04 13:30:00 +1000
---
Ever been on a project where a dev comes on board, and has to clone databases in order to get test data?  What about when you just want to
nuke all your test data and start afresh - is starting afresh pretty painful?

We went whole-hog on seed data and test user generation recently, found it to be _incredibly_ useful, and will be doing it on future projects.

This post covers the fake user creation aspect.

<!--break-->

## Users

When we first rolled in, the authentication story was up in the air, but a lot of the features relied on being an authenticated party.
Rolling a test [IdentityServer](https://github.com/IdentityServer/IdentityServer3) was the way to go, but filling it up with users 
seemed laborious.

And I'm a lazy programmer.

So I generated them! There we some minimal bits of information we needed per user:

* Id (guid)
* Name
* Email
* Role

What worked for us was generating data which matched the domain representation of our users, and then mapping them to an in memory
store of users.

So, generating customers using [Bogus](https://github.com/bchavez/Bogus) looks like this:

```cs
public class CustomerInformation
{
    public Guid Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public string Email { get; set; }
    public string Country { get; set; }
}

public static class FakeCustomers
{
    private const int NumCustomers = 100;

    private static readonly Guid CustomerNameSpace = Guid.Parse("0F6C5C66-C102-4F77-94C6-C772813F21F6");
    private static readonly Faker<CustomerInformation> CustomerFaker = new Faker<CustomerInformation>()
        .StrictMode(true)
        .RuleFor(c => c.Id, f => GuidUtility.Create(CustomerNameSpace, f.Random.AlphaNumeric(20)))
        .RuleFor(c => c.FirstName, f => f.Name.FirstName())
        .RuleFor(c => c.LastName, f => f.Name.LastName())
        .RuleFor(c => c.Email, (f, c) => $"{c.FirstName}.{c.LastName}@example.com")
        .RuleFor(c => c.Country, f => f.Address.Country());

    static FakeCustomers()
    {
        var random = new Random(1);
        Randomizer.Seed = random;

        All = CustomerFaker.Generate(NumCustomers).ToList();
    }

    public static readonly IReadOnlyCollection<CustomerInformation> All;
}
``` 

This uses a handy [Guid Utility](https://github.com/LogosBible/Logos.Utility/blob/master/src/Logos.Utility/GuidUtility.cs) detailed
[on this blog post](https://code.logos.com/blog/2011/04/generating_a_deterministic_guid.html) to create namespaced deterministic guids.

Note that we set the initial random seed - we want to ensure multiple runs produces the same data, the consistency is very helpful.

This approach works well because the shared customer data can be pulled into a separate project and used in tests and seed data generation.

After customers are created, we set up our Identity Server to use them:

```cs
public static class Users
{
    public static IEnumerable<InMemoryUser> All
    {
        get 
        {
            return FakeCustomers.All.Select(c => CreateUser(c.Id, c.FirstName, c.LastName, c.Email, "Customer"))
        }
    }

    private static CreateUser(Guid id, string firstName, string lastName, string email, string role) 
    {
        string username = $"{firstName}.{lastName}";

        var claims = new List<Claim>(new[]
        {
                new Claim(Constants.ClaimTypes.Subject, username),
                new Claim(Constants.ClaimTypes.Id, id.ToString("D").ToUpperInvariant()),
                new Claim(Constants.ClaimTypes.Email, email),
                new Claim(Constants.ClaimTypes.GivenName, firstName),
                new Claim(Constants.ClaimTypes.FamilyName, lastName),
                new Claim(Constants.ClaimTypes.Role, role),
        });

        return new InMemoryUser
        {
            Subject = username,
            Username = username,
            Password = "test",
            Enabled = true,
            Claims = claims.ToArray()
        };
    }
}
```

And at startup:

```cs
var factory = new IdentityServerServiceFactory();

factory
    .UseInMemoryClients(/*as required*/)
    .UseInMemoryScopes(/*as required*/)
    .UseInMemoryUsers(Users.All.ToList());
```

Finally, hack some markup so you don't have to remember the login details. Since this is for dev only, it doesn't matter!

In `templates/_login.html`:

```html
<h3 style="clear: both;">Customer</h3>
<form ng-repeat="user in model.custom.customers | limitTo:5" method="post" action="{{model.loginUrl}}" class="login-button-form">
    <anti-forgery-token token="model.antiForgery"></anti-forgery-token>

    <input type="hidden" name="username" value="{{user.username}}"/>
    <input type="hidden" name="password" value="test"/>

    <input type="hidden" name="rememberMe" value="true">
    <div class="form-group">
        <input type="submit" value="{{ user.display }}" class="button"/>
    </div>
</form>
```

In a `CustomViewService`:

```cs
model.Custom = new
{
    customers = FakeCustomers.All.Select(c => new
    {
        username = $"{c.FirstName}.{c.LastName}",
        display = $"{c.FirstName} {c.LastName}"
    }).ToArray()
}
```

These conventions match our user generation, and a form per user means single-button login for everything.

![Login sample](/images/2016-01-04-fake-login.png)

What a nice dev login experience we have.

We also did a small amount of codegen to "hardcode" some of the users by name, by generating a static class like:

```cs
public static class TestUsers {
    public static class Customers {
        public static CustomerInformation JohnSmith => FakeCustomers.All.Skip(0).First();
        public static CustomerInformation MaryJane => FakeCustomers.All.Skip(1).First();
    }
}
```

Meaning in a test, we can go

```cs
var customer = TestUsers.Customers.JohnSmith;
```

Which is more handy than it sounds.

This codegen is relatively easy, if you want to go that route.  We found we didn't use many different users, and since our usernames
don't change this is probably simplest to just hardcode.

All in all, this approach worked fantastically - we set up relationships between users and groups using the same approach,
and not having to worry about remember user names was great.  Especially during our sprint reviews, being able to easily sign in
and out of different roles as our stakeholders wanted to see different bits was a life saver.

Stay tuned for the next part - exercising our API by generating seed data.