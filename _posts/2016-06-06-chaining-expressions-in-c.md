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
More and more I've been using projections to handle the _query_ side of my applications, which of course includes a lot of `Expression` objects.

The problem with `Expression` objects is they're non-trivial to chain together and combine, because they're data structures, not code.

I recently had to implement a simple report filter that had optional date ranges on 4 different date fields - each with an optional **From** and **To** date.  Of course, there were other requirements of this feature, too, which makes it a bit more interesting:

* Run the whole query in SQL - I don't want to materialize my enumerable to perform the filtering
* Somewhat simple SQL query - of course I can be clever with `GroupBy` and `SelectMany`, but I'd prefer my SQL to just say `WHERE date <= @p0` if possible.
* Clean code - DRY, reusable, terse _et. al._

<!--break-->
This is a faithful reproduction of my filter:

```c#
public class CarFilter
{
    public DateTime? RegistrationDateFrom { get; set; }
    public DateTime? RegistrationDateTo { get; set; }
    
    public DateTime? PurchaseDateFrom { get; set; }
    public DateTime? PurchaseDateTo { get; set; }
}
```

When I design tthings that I want to be highly readable, I generally write the calling code first:

```c#
public IEnumerable<Car> GetCars(CarFilter carFilter)
{
    return carFilter.ApplyTo(_carQuery);
}
```

See how clean that is? Ok ok, so that was cheating....

```c#
public IQueryable<Car> ApplyTo(IQueryable<Car> carQuery)
{
    return carQuery
        .WhereDateBetween(car => car.RegistrationDate, RegistrationDateFrom, RegistrationDateTo)
        .WhereDateBetween(car => car.PurchaseDate, PurchaseDateFrom, PurchaseDateTo);
}
```

I thought an extension method was better suited for the example here, but the real solution used a seperate Query object. Just so ya know.

```c#
public static IQueryable<T> WhereDateBetween<T>(this IQueryable<T> source, 
    Expression<Func<T, DateTime>> getDate, 
    DateTime? fromDate DateTime? toDate)
{
    if (from == null && to == null)
        return source; // The simplest query is no query

    // Uhhh...
}
```

At this point I thought I'd pull out a predicate expression - that is `Expression<Func<DateTime, bool>>`.

```c#
private static Expression<Func<DateTime, bool>> DateBetween(DateTime? fromDate, DateTime? toDate)
{
    if (toDate == null)
        return date => fromDate <= date;

    if (fromDate == null)
        return date => toDate >= date;

    return date => fromDate <= date && toDate >= date;
}
```

I do love that `date => fromDate <= date` is perfectly valid code.

Essentially what we have now are 2 expressions: `GetDate(T) : DateTime` and `Predicate(Date) : bool`. What we _want_ is an expression that represents `Predicate(GetDate(T)) : bool`.  Thus our `WhereDateBetween` function will become:

```c#
public static IQueryable<T> WhereDateBetween<T>(this IQueryable<T> source, 
    Expression<Func<T, DateTime>> getDate, 
    DateTime? fromDate DateTime? toDate)
{
        if (fromDate == null && toDate == null)
        return source; // The simplest query is no query

        var predicate = DateBetween(fromDate, toDate);
        return source.Where(getDate.Chain(predicate));
}
```

If you don't want to understand expression trees, then implementing `Chain` can be done by [copy/pasting from stack overflow](http://stackoverflow.com/questions/7873448/create-dynamic-expression-lambda-from-two-others-chaining-the-expressions) and attributing appropriately. Unfortunately the code is so straight forward it's going to look like that's what I did, but I swear I found this _afterwards_ - my GoogleFu was weak in this instance.

Before we move on, let's get a bit more generic and mathematical. We have 2 expressions, `F(a)` and `G(b)`, and want one expression, `G(F(a))`.  This can be done by replacing the parameter `b` of `G` with the body of `F`.

The simplest way to replace parts of expression trees is to use the `ExpressionVisitor` base class that was introduced back in .Net 3.5.  It just provides a bunch of methods that you can override to substitute parts of trees from others. In our case, we're just need to swap one node with another.

```c#
internal class SwapVisitor : ExpressionVisitor
{
    private readonly Expression _source, _replacement;

    public SwapVisitor(Expression source, Expression replacement)
    {
        _source = source;
        _replacement = replacement;
    }

    public override Expression Visit(Expression node)
    {
        return node == _source ? _replacement : base.Visit(node);
    }
}
```

And then implement the `Chain` method

```c#
public static Expression<Func<TIn, TOut>> Chain<TIn, TInterstitial, TOut>(
    this Expression<Func<TIn, TInterstitial>> inner,
    Expression<Func<TInterstitial, TOut>> outer)
{
    var visitor = new SwapVisitor(outer.Parameters[0], inner.Body);
    return Expression.Lambda<Func<TIn, TOut>>(visitor.Visit(outer.Body), inner.Parameters);
}
```

This creates a `SwapVisitor` that replaces the first parameter of `outer` (the `b` in `G(b)`) with the body of `inner` (the `F` in `F(a)`). It then creates the new lambda expression with the body `G∘F` (wiki: [function composition](https://en.wikipedia.org/wiki/Function_composition)) and the parameter from `inner` (the `a` of `F(a)`).

And we're done! Very simple, type safe code (except for the expression tree twiddling) that produces nice, fast queries!

I've uploaded a [gist of all the implementing files](https://gist.github.com/xwipeoutx/962b205324017c000c75899a8b5016d9) if you want to see it all without distractions.