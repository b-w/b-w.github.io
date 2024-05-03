---
layout: post
title: "Entity Framework conditional includes"
categories: blog
---

If you've ever worked with Entity Framework, then you're probably familiar with the [.Include(...) function](https://msdn.microsoft.com/en-us/library/jj574232(v=vs.113).aspx). You've probably also wondered if it's possible to perform a _conditional_ include (i.e. including only those entities that match a certain predicate). The include function itself doesn't allow for this, and I thought it simply wasn't possible until a coworker showed me the following trick.

Let's say we have a simple model:

```csharp
public class Product
{
    public int Id { get; set; }

    public DateTimeOffset InStock { get; set; }

    public DateTimeOffset? OutOfStock { get; set; }

    public virtual ICollection<ProductResource> Resources { get; set; }
}

public class ProductResource
{
    public int ProductId { get; set; }

    public string LanguageCode { get; set; }

    public string Name { get; set; }

    public string Description { get; set; }
}
```

Here we have a product, which contains a set of resources in multiple languages. Not an uncommon scenario.

We can imagine a simple repository function like this:

```csharp
public Product GetProduct(int id)
{
    return _context.Products
                .Include(x => x.Resources)
                .Where(x => x.Id == id)
                .FirstOrDefault();
}
```

This'll retrieve a requested product, and its resources. _All_ of its resources.

If we use it like this:

```csharp
static void Main(string[] args)
{
    using (var ctx = new ProductContext())
    {
        var repo = new ProductRepository(ctx);
        var product = repo.GetProduct(1);
        Print(product);
    }
}

static void Print(Product product)
{
    Console.WriteLine($"Product Id={product.Id}");
    foreach (var resource in product.Resources)
    {
        Console.WriteLine($"[{resource.LanguageCode}] {resource.Name}: {resource.Description}");
    }
}
```

We get the following result, as expected:

```
    Product Id=1
    [EN] Tea: Some delicious tea.
    [NL] Thee: Overheerlijke thee.
```

In many cases an include like that will be fine, or even intended. But in our example, we only really need the resources for one language, because that's all we need to show to the user. Blindly retrieving all resources, for potentially dozens of languages, is wasteful.

So here's where conditional includes would be welcome. We may not be able to use the include function for this, but we can do the following:

```csharp
public Product GetProduct(int id, string languageCode)
{
    return _context.Products
                .Where(x => x.Id == id)
                .Select(x => new
                {
                    p = x,
                    r = x.Resources.Where(y => y.LanguageCode.Equals(languageCode))
                })
                .AsEnumerable()
                .Select(x => x.p)
                .FirstOrDefault();
}
```

This causes Entity Framework to generate the following SQL query:

```sql
SELECT 
    [Project1].[Id] AS [Id], 
    [Project1].[InStock] AS [InStock], 
    [Project1].[OutOfStock] AS [OutOfStock], 
    [Project1].[C1] AS [C1], 
    [Project1].[ProductId] AS [ProductId], 
    [Project1].[LanguageCode] AS [LanguageCode], 
    [Project1].[Name] AS [Name], 
    [Project1].[Description] AS [Description]
    FROM ( SELECT 
        [Extent1].[Id] AS [Id], 
        [Extent1].[InStock] AS [InStock], 
        [Extent1].[OutOfStock] AS [OutOfStock], 
        [Extent2].[ProductId] AS [ProductId], 
        [Extent2].[LanguageCode] AS [LanguageCode], 
        [Extent2].[Name] AS [Name], 
        [Extent2].[Description] AS [Description], 
        CASE WHEN ([Extent2].[ProductId] IS NULL) THEN CAST(NULL AS int) ELSE 1 END AS [C1]
        FROM  [dbo].[Product] AS [Extent1]
        LEFT OUTER JOIN [dbo].[ProductResource] AS [Extent2]
            ON ([Extent1].[Id] = [Extent2].[ProductId]) AND ([Extent2].[LanguageCode] = @p__linq__1)
        WHERE [Extent1].[Id] = @p__linq__0
    ) AS [Project1]
    ORDER BY [Project1].[Id] ASC, [Project1].[C1] ASC
```

Note that in the join with the ProductResource table, the language code is included as part of the join condition.

If we run this:

```csharp
static void Main(string[] args)
{
    using (var ctx = new ProductContext())
    {
        var repo = new ProductRepository(ctx);
        var product = repo.GetProduct(2, "EN");
        Print(product);
    }
}
```

We get the expected result:

```
Product Id=2
[EN] Biscuits: Delightful biscuits, old bean!
```

This works as expected, saves us some resources, and is a nice way of performing a conditional include without having to manually write our own joins in the Linq query syntax. What's not to love?

Well... there's a very important catch. Or should I say, _cache_? You see, if we do this:

```csharp
static void Main(string[] args)
{
    using (var ctx = new ProductContext())
    {
        var repo = new ProductRepository(ctx);

        var product1 = repo.GetProduct(2, "EN");
        Print(product1);

        var product2 = repo.GetProduct(2, "NL");
        Print(product2);
    }
}
```

We get _this_:

```
Product Id=2
[EN] Biscuits: Delightful biscuits, old bean!
Product Id=2
[EN] Biscuits: Delightful biscuits, old bean!
[NL] Koekjes: Waanzinnige koekjes, oude boon!
```

Not what we were expecting! The second product includes both languages, when we only asked for Dutch.

So what went wrong? Well, it's not the SQL query. Looking at the profiler, we can see that it correctly requests just the Dutch resource to be joined. The answer is found in Entity Framework's own internal cache. Because the English resource is already present in the cache (from the previous query), it simply gets included any time the product with Id 2 gets retrieved. Whether we ask for it or not.

This is extremely important to keep in mind when working with these kind of conditional includes, because I can easily see this being the cause of many head-scratching bugs otherwise.
