---
layout: post
title: Entity Framework Core range query extension
---

So earlier in the week I was looking at refactoring some older code in our system which added a whole lot of range queries using Entity Framework.

In short, the code looked like

{% highlight cs %}
switch(searchOperator)
{
    case ">=":
        query = query.Where(x => x.Property >= foo);
        break;
    case " x.Property <= foo);
        break;
    // ... cont
}
{% endhighlight %}

You can imagine the above code repeated 5 or 6 times for various dates and values causing a rather bloated class with lots of repeated similar logic.

Not ideal and after a bit of searching there isn’t much guidance for how to maintain common logic like this.

After some trial and error I managed to come up with an extension method to encapsulate the repeated logic.

Firstly I wrapped each property in a generic object.

{% highlight cs %}
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

I should mentioned SearchOperator is an enum local to my project.

The search model passed to my repository now has a richer set of properties for searching. This eliminates the repeated use of fields such as RateValue and RateSearchOperator.

{% highlight cs %}
public class SearchCriteria
{
    public Guid BorrowingParty { get; set; }
    public RangeSearch Created { get; set; } 
    public RangeSearch Value { get; set; } 
}
{% endhighlight %}

And this is where the magic happens.

Now this could probably be tidied up to better fit your requirements but for my simple case, this worked a treat.

{% highlight cs %}
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

So lastly we need to put it all together.

{% highlight cs %}
public Task SearchAsync(SearchCriteria searchCriteria)
{
    return _context
        .LoanSummary
        .Where(s => s.BorrowingParty == searchCriteria.BorrowingParty)
        .WhereRangeSearch(s => s.LoanCreatedDate, searchCriteria.Created)
        .WhereRangeSearch(s => s.Quantity * s.SecurityPrice, searchCriteria.Value);
        .ToListAsync();     
}
{% endhighlight %}

This solution worked great for my team’s project and we managed to eliminate a few hundred lines of code in the process!

The generated SQL looks pretty tidy too 😀

— Dan