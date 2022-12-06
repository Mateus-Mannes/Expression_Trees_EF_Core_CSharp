# Using C# Expression Trees to create a custom database query with EF Core
The EF Core is a ORM and free us from write and maintain SQL querys into our source code and from caring about translating the querys results to our projects entities. However, sometimes we have problems on translating a LINQ query that we do on a DBSet/Repository to the desired SQL query, resulting on an error like this:

```bash
System.InvalidOperationException: The LINQ expression '...' could not be translated. Either rewrite the query in a form that can be translated, or switch to client evaluation explicitly by inserting a call to 'AsEnumerable', 'AsAsyncEnumerable', 'ToList', or 'ToListAsync
```

And of course, there are expressions that can’t be translated by the EF Core, but the reason of the error can maybe just be a bad structuration of the expression tree.

## Expression Trees

An Expression Tree is a representation of a series of commands connected into a structure that looks like a tree. It needs to be built from bottom to top and its nodes are represented by the class Expression from C#. To clarify, lets see an example of how we can build a simple lambda expression using an Expression Tree:

Lambda:

```C#
Func<int, bool> LessThanFive = n => n < 5;
```

Building the Expression:

```C#
using System.Linq.Expressions;

var parameter = Expression.Parameter(typeof(int));

Expression<Func<int, bool>> LessThanFiveExpression = 
    Expression.Lambda<Func<int, bool>>(
        Expression.LessThan(
            parameter,
            Expression.Constant(5)
        ), parameter
    );

var LessThanFive = LessThanFiveExpression.Compile();
```

At the end, this would be the visual representation of the tree:

![image](https://user-images.githubusercontent.com/64140337/205742749-005f308a-37a2-4ab4-93c1-cf086dd59a2d.png)

Knowing about this structure is important because, when we execute something like this:

```C#
var list = await _context.Entity.Where(x => x.Property == "value").ToListAsync();
```

The EF Core uses the Tree structure that represents the filter "`x => x.Property == "value"`" to work as a compiler and generate the adequate SQL code as it iterates node by node.

## Example of a real case of use

### The problem

I was working with this type of query:

```C#
var result = await _context.Entity.Where(x => list.Contains(x.Property)).ToListAsync();
```

That is translated by the EF Core as:

```SQL
SELECT * FROM ENTITY WHERE PROPERTY IN ( ... )
```

However, the Oracle client has a limit of 1000 items that can be passed into the `IN` clause. When this limit is passed over, the result is an error like: 

```bash
ORA-01795: maximum number of expressions in a list is 1000 error
```

A possible solution to this is to split the list into lists with the maximum of 1000 items. Creating a function to do the split, the code would be something like this:

```C#
var splitedList = SplitList(filter, 1000);
var result = new List<Entity>();
foreach (var list in splitedList)
    result.AddRange(await _context.Entity.Where(x => list.Contains(x.Property)).ToListAsync());
```

This way, the initial problem is resolved. But what if we need to execute this kind of query in different parts of the solution, different tables and filtering by different columns? We would need to repeat this same logic, repeating these 5 lines of code as it is needed. Therefore, a generic method for this would make our code a little better.

A first try using an Extension Method should looks like this:

```C#
public async static Task<List<T>> SelectInChunksAsync<T, U>(this IQueryable<T> repository, List<U> listFilter, Func<T, U> selector)
{
    var splitedList = SplitList(listFilter, 1000);
    var result = new List<T>();
    foreach (var list in splitedList)
        result.AddRange(await repository.Where(x => list.Contains(selector(x))).ToListAsync());
    return result;
}
```

Note that the parameter `selector` is a lambda function that returns the value of the property I am filtering. Executing this method in a list that is already loaded into the memory runs nicely, but running this on a  DbSet turns into that translation error:

```bash
System.InvalidOperationException: The LINQ expression '...' could not be translated. Either rewrite the query in a form that can be translated, or switch to client evaluation explicitly by inserting a call to 'AsEnumerable', 'AsAsyncEnumerable', 'ToList', or 'ToListAsync
```

This happens because the lambda `selector` can’t be translated this way, been inserted directedly into the Expression of the `Where()` clause.

Therefore, a possible solution is to build the `where` predicate from scratch, using the Expression class, and building the parameters and operations in the correct order, so then the EF Core would be able to read it: 

```C#
public async static Task<List<T>> SelectInChunksAsync<T, U>(this IQueryable<T> repository, List<U> listFilter, Expression<Func<T, U>> selector)
{
    var splitedList = SplitList(listFilter, 1000);
    var result = new List<T>();
    foreach(var list in splitedList) 
        result.AddRange(await repository.SelectInAsync(list, selector));
    return result;
}

private async static Task<List<T>> SelectInAsync<T, U>(this IQueryable<T> repository, List<U> listFilter, Expression<Func<T, U>> selector)
{
    // Constant Expression for the list filter
    var listaConstante = Expression.Constant(listFilter, listFilter.GetType());

    // Entity parameter for the query
    var parameter = Expression.Parameter(typeof(T));

    // Expression to access the property from the selector
    var memberExpression = (MemberExpression)selector.Body;
    var accessor = Expression.MakeMemberAccess(parameter, memberExpression.Member);

    // Calling for the Contains method
    var callContains = Expression.Call(typeof(Enumerable), "Contains", new Type[] { accessor.Type }, listaConstante, accessor);
    var predicate = Expression.Lambda<Func<T, bool>>(callContains, parameter);

    return await repository.Where(predicate).ToListAsync();
}
```

Note that now we are instantiating the parameter `selector` as `Expression<Func<T, U>>`, what able us to manipulate it as a Expression Tree. Furthermore, we added the method `SelectInAsync`, that is now responsible for building the predicate and executing the query. Note also that we are working with different types of Expressions, Constant, Parameter, Member, MethodCall (Contains), each one has its own responsibility on the chain of commands.

Running it on debug mode and analyzing the variables, the value of the final filter `predicate` is:

```C#
{Param_0 => value(System.Collections.Generic.List`1[Type]).Contains(Param_0.Property)}
```
Therefore, now we have a generic way of executing the search querys in parts, doing this request:

```C#
var result = await _context.DbSet.SelectInChunksAsync(filterList, x => x.ColumnToFilter);
```

A SQL representation, as if we had different lists of values to filter in the same select statement:

```SQL
SELECT * FROM TABLE WHERE COLUMN IN ( ... )
UNION ALL 
SELECT * FROM TABLE WHERE COLUMN IN ( ... )
UNION ALL 
SELECT * FROM TABLE WHERE COLUMN IN ( ... )
...
```

I believe that knowing about Expression Trees is a very powerful knowledge. Here is the resources I used to write this content:

- Microsoft documentation about Expression Trees:
    - [Building Expression Trees](https://learn.microsoft.com/en-us/dotnet/csharp/expression-trees-building)
    - [Expression Trees (C#)](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/expression-trees/)

- Shay Rojansky presentation, very very interesting: 
    - [Shay Rojansky - How Entity Framework translates LINQ all the way to SQL - Dotnetos Conference 2019](https://www.youtube.com/watch?v=r69ZxXgOIK4)
