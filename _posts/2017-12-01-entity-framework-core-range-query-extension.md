---
layout: post
title: Entity Framework Core range query extension
---

Earlier this the week I was refactoring some search logic in our system which added a whole heap of range queries using **Entity Framework**.

In short, the code looked like

{% highlight csharp %}
switch(searchOperator)
{
    case "==":
        query = query.Where(x => x.Property == foo);
        break;
    case ">=":
        query = query.Where(x => x.Property >= foo);
        break;
    case "<=":
        query = query.Where(x => x.Property <= foo);
        break;
}
{% endhighlight %}

You can imagine the above code repeated 5 or 6 times for various dates and values caused a rather bloated class with lots of similar
core search logic. Code duplication like this is not ideal and I find it can be fairly fragile.

After a bit of searching I didn't find much guidance for common search logic like this so I decided to hand crank my own!

To start with I created a generic object which encapsulates the property, it's search value and it's search operator.
The search criteria model I pass to my repository now has a richer set of properties for searching. 

This has also eliminated the repeated use of properties such as `nValue` and `nSearchOperator`.

{% highlight csharp %}
public class SearchCriteria
{
    public Guid BorrowingParty { get; set; }
    public RangeSearch Created { get; set; } 
    public RangeSearch LoanValue { get; set; } 
}

public enum SearchOperator
{
    EqualTo,
    GreaterThanOrEqualTo,
    LessThanOrEqualTo
}

public class RangeSearch
{
    public SearchOperator? SearchOperator { get; }
    public TValue Value { get; }

    public RangeSearch()
    {
    }

    public RangeSearch(SearchOperator searchOperator, TValue value)
    {
        SearchOperator = searchOperator;
        Value = value;
    }

    public bool HasValue => SearchOperator != null;
}
{% endhighlight %}

Now to finally tackle the mighty switch statement above.

This could probably be tidied up to better fit your requirements but for my simple case, this Entity Framework
extension works a treat.

{% highlight csharp %}
public static IQueryable WhereRangeSearch(this IQueryable source, Expression selector, RangeSearch rangeSearch)
{
    if (!rangeSearch.HasValue)
        return source;
    
    var parameter = selector.Parameters.Single();
    BinaryExpression expression;

    switch (rangeSearch.SearchOperator)
    {
        case SearchOperator.EqualTo:
            expression = Expression.Equal(selector.Body, Expression.Constant(rangeSearch.Value, typeof(TResult)));
            break;

        case SearchOperator.GreaterThanOrEqualTo:
            expression = Expression.GreaterThanOrEqual(selector.Body, Expression.Constant(rangeSearch.Value, typeof(TResult)));
            break;

        case SearchOperator.LessThanOrEqualTo:
            expression = Expression.LessThanOrEqual(selector.Body, Expression.Constant(rangeSearch.Value, typeof(TResult)));
            break;
            
        default:
            throw new ArgumentOutOfRangeException();
    }
    
    return source.Where(Expression.Lambda(expression, parameter));
}
{% endhighlight %}

And lastly lets put it all together.

{% highlight csharp %}
public Task SearchAsync(SearchCriteria searchCriteria)
{
    return _context
        .LoanSummary
        .Where(s => s.BorrowingParty == searchCriteria.BorrowingParty)
        .WhereRangeSearch(s => s.LoanCreatedDate, searchCriteria.Created)
        .WhereRangeSearch(s => s.Quantity * s.SecurityPrice, searchCriteria.LoanValue);
        .ToListAsync();
}
{% endhighlight %}

This solution has worked great for my team’s project and we managed to eliminate a few hundred lines of code in the process!

Entity framework core generates some pretty tidy looking SQL too :D

&mdash; Dan