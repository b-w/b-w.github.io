---
layout: post
title: "Building Blazer, part 2"
categories: blog
---

We continue work on Blazer, our high-performance ADO.NET wrapper.

## Parameter type mapping

Last time we worked on the parameter mapper, but our code still had a single todo in it: we had to map types of input values to their respective DbType values. I've decided to solve this using a simple hardcoded map:

```csharp
namespace Blazer
{
    using System;
    using System.Collections.Concurrent;
    using System.Data;

    internal static class DbTypeMap
    {
        static readonly ConcurrentDictionary<Type, DbType> m_typeMap = new ConcurrentDictionary<Type, DbType>()
        {
            [typeof(string)] = DbType.String,
            [typeof(char)] = DbType.StringFixedLength,
            [typeof(bool)] = DbType.Boolean,
            [typeof(byte)] = DbType.Byte,
            [typeof(sbyte)] = DbType.SByte,
            [typeof(short)] = DbType.Int16,
            [typeof(ushort)] = DbType.UInt16,
            [typeof(int)] = DbType.Int32,
            [typeof(uint)] = DbType.UInt32,
            [typeof(long)] = DbType.Int64,
            [typeof(ulong)] = DbType.UInt64,
            [typeof(float)] = DbType.Single,
            [typeof(double)] = DbType.Double,
            [typeof(decimal)] = DbType.Decimal,
            [typeof(Guid)] = DbType.Guid,
            [typeof(DateTime)] = DbType.DateTime,
            [typeof(DateTimeOffset)] = DbType.DateTimeOffset,
            [typeof(TimeSpan)] = DbType.Time,
            [typeof(object)] = DbType.Object,
            [typeof(byte[])] = DbType.Binary
        };

        public static bool TryGetDbType(Type forType, out DbType dbType)
        {
            var type = forType;
            var nullableType = Nullable.GetUnderlyingType(type);
            if (nullableType != null)
            {
                type = nullableType;
            }
            if (type.IsEnum && !m_typeMap.ContainsKey(type))
            {
                type = Enum.GetUnderlyingType(type);
            }

            return m_typeMap.TryGetValue(type, out dbType);
        }

        public static void RegisterType(Type type, DbType dbType)
        {
            m_typeMap.AddOrUpdate(type, dbType, (key, oldValue) => dbType);
        }
    }
}
```

The map is used in our parameter expression factory:

```csharp
public static Expression GetExpression(Context context, PropertyInfo property)
{
    DbType dbType;
    if (DbTypeMap.TryGetDbType(property.PropertyType, out dbType))
    {
        return GetExpressionForKnownDbType(context, property, dbType);
    }

    if (typeof(IEnumerable<>).IsAssignableFrom(property.PropertyType))
    {
        var innerType = property.PropertyType.GenericTypeArguments[0];
        if (DbTypeMap.TryGetDbType(innerType, out dbType))
        {
            // TODO: handle collection parameter
        }

        throw new NotSupportedException($"Collection parameter of type {innerType} is not supported. Only collections of known data types are supported.");
    }

    throw new NotSupportedException($"Parameter of type {property.PropertyType} is not supported.");
}
```

I've also done some light refactoring here. The original factory function is now called GetExpressionForKnownDbType, and is called from the GetExpression function. The former function produces mapping expressions for simple value properties, but in the latter I'm also planning support for collection parameters. Speaking of which...

## Collection parameters

Although I want to keep Blazer as simple as possible, without any black box ORM magic, I still want to support collection parameters. Mainly because I feel that this is something that'll be used a lot, and is quite tedious if you have to do the required work yourself. So, Blazer'll take care of it.

So, what are collection parameters? Consider the following SQL query:

```sql
SELECT *
FROM Products
WHERE CategoryId IN (@Categories)
```

Although it might look that way, in ADO.NET you can't simply add an array parameter here and expect this to work. You'll have to do something like this:

```sql
SELECT *
FROM Products
WHERE CategoryId IN (@Cat1, @Cat2, @Cat3)
```

...and then add a parameter for each category id. This is tedious and error-prone, so in Blazer I want to automate this. When passing a collection parameter, I want Blazer to automatically replace the parameter in the query string, and add parameters for each value in the collection. For this purpose, the parameter factory should build the following expression:

```csharp
var sb = new StringBuilder();
sb.Append("(");

var i = 0;
foreach (var <item> in <collection>)
{
    if (i > 0)
    {
        sb.Append(",");
    }

    sb.Append("<prop_name>");
    sb.Append(i);

    var dataParam = command.CreateParameter();
    dataParam.Direction = ParameterDirection.Input;
    dataParam.DbType = <some_type>;
    dataParam.ParameterName = "@<prop_name>" + i;
    dataParam.Value = <item>;

    command.Parameters.Add(dataParam);

    i++;
}

sb.Append(")");

var regexPattern = @"\(\s*(" + Regex.Escape("@<prop_name>") + @")\s*\)";
command.CommandText = Regex.Replace(command.CommandText, regexPattern, sb.ToString());
```

So for example, if we were to run that code on the following query:

```sql
SELECT *
FROM Products
WHERE CategoryId IN (@Categories)
```

...and we pass it an integer array `[2, 7, 16]`, then the query text will be transformed into:

```sql
SELECT *
FROM Products
WHERE CategoryId IN (@Categories1, @Categories2, @Categories3)
```

...and the following parameters will be added:

```sql
("Categories1", 2)
("Categories2", 7)
("Categories3", 16)
```

In Blazer, we'll generate these expressions at runtime using the following factory method:

```csharp
static Expression GetExpressionForCollectionType(Context context, PropertyInfo property, Type collectionType, DbType dbType)
{
    var propertyVariableName = "@" + property.Name;

    var sbType = typeof(StringBuilder);
    var sbAppendStringMethod = sbType.GetMethod("Append", new[] { typeof(string) });
    var sbAppendIntMethod = sbType.GetMethod("Append", new[] { typeof(int) });
    var sbToStringMethod = sbType.GetMethod("ToString", new Type[] { });
    var stringConcatMethod = typeof(string).GetMethod("Concat", new[] { typeof(object), typeof(object) });
    var rxEscapeMethod = typeof(Regex).GetMethod("Escape", new[] { typeof(string) });
    var rxReplaceMethod = typeof(Regex).GetMethod("Replace", new[] { typeof(string), typeof(string), typeof(string) });

    // StringBuilder sb;
    var sbVarExpr = Expression.Variable(sbType);

    // sb = new StringBuilder();
    var sbNewExpr = Expression.Assign(
        sbVarExpr,
        Expression.New(sbType));

    // sb.Append("(");
    var sbOpenBracketExpr = Expression.Call(
        sbVarExpr,
        sbAppendStringMethod,
        Expression.Constant("("));

    // int i;
    var iVarExpr = Expression.Variable(typeof(int));

    // i = 0;
    var iInitExpr = Expression.Assign(iVarExpr, Expression.Constant(0));

    // foreach (var <item> in <collection>)
    // {
    var loopVarExpr = Expression.Variable(collectionType);

    //      if (i > 0)
    //      {
    //          sb.Append(",");
    //      }
    var sbAppendCommaExpr = Expression.IfThen(
        Expression.GreaterThan(iVarExpr, Expression.Constant(0)),
        Expression.Call(
            sbVarExpr,
            sbAppendStringMethod,
            Expression.Constant(",")));

    //      sb.Append("<prop_name>");
    var sbAppendPropNameExpr = Expression.Call(
        sbVarExpr,
        sbAppendStringMethod,
        Expression.Constant(propertyVariableName));

    //      sb.Append(i);
    var sbAppendIExpr = Expression.Call(
        sbVarExpr,
        sbAppendIntMethod,
        iVarExpr);

    //      IDbDataParameter dataParam;
    var dbParamVarExpr = Expression.Variable(typeof(IDbDataParameter));

    //      dataParam = command.CreateParameter();
    var createParamExpr = Expression.Assign(
        dbParamVarExpr,
        Expression.Call(context.CommandExpr, context.CreateParamMethod));

    //      dataParam.Direction = ParameterDirection.Input;
    var directionExpr = Expression.Assign(
        Expression.Property(dbParamVarExpr, context.ParamDirectionProperty),
        Expression.Constant(ParameterDirection.Input));

    //      dataParam.DbType = <some_type>;
    var dbTypeExpr = Expression.Assign(
        Expression.Property(dbParamVarExpr, context.ParamDbTypeProperty),
        Expression.Constant(dbType));

    //      dataParam.ParameterName = "@<prop_name>" + i;
    var nameExpr = Expression.Assign(
        Expression.Property(dbParamVarExpr, context.ParamNameProperty),
        Expression.Call(
            stringConcatMethod,
            Expression.Constant(propertyVariableName),
            Expression.Convert(iVarExpr, typeof(object))));

    //      dataParam.Value = <item>;
    var valueExpr = Expression.Assign(
        Expression.Property(dbParamVarExpr, context.ParamValueProperty),
        Expression.Convert(
            loopVarExpr,
            typeof(object)));

    //      command.Parameters.Add(dataParam);
    var addParamExpr = Expression.Call(
        Expression.Property(context.CommandExpr, context.CommandParametersProperty),
        context.CommandParametersAddMethod,
        dbParamVarExpr);

    //      i++;
    var iIncrementExpr = Expression.Increment(iVarExpr);

    // }
    var loopExpr = ExpressionHelper.ForEach(
        loopVarExpr,
        Expression.Property(context.ParametersExpr, property),
        Expression.Block(
            new[] { dbParamVarExpr },
            sbAppendCommaExpr,
            sbAppendPropNameExpr,
            sbAppendIExpr,
            createParamExpr,
            directionExpr,
            dbTypeExpr,
            nameExpr,
            valueExpr,
            addParamExpr,
            iIncrementExpr
            ));

    // sb.Append(")");
    var sbCloseBracketExpr = Expression.Call(
        sbVarExpr,
        sbAppendStringMethod,
        Expression.Constant(")"));

    // string regexPattern;
    var rxVarExpr = Expression.Variable(typeof(string));

    // regexPattern = @"\(\s*(" + Regex.Escape("@<prop_name>") + @")\s*\)";
    var rxAssignExpr = Expression.Assign(
        rxVarExpr,
        Expression.Call(
            stringConcatMethod,
            Expression.Constant(@"\(\s*("),
            Expression.Call(
                stringConcatMethod,
                Expression.Call(
                    rxEscapeMethod,
                    Expression.Constant(propertyVariableName)),
                Expression.Constant(@")\s*\)"))));

    // command.CommandText = Regex.Replace(command.CommandText, regexPattern, sb.ToString());
    var cmdTextAssignExpr = Expression.Assign(
        Expression.Property(context.CommandExpr, context.CommandCommandTextProperty),
        Expression.Call(
            rxReplaceMethod,
            Expression.Property(context.CommandExpr, context.CommandCommandTextProperty),
            rxVarExpr,
            Expression.Call(
                sbVarExpr,
                sbToStringMethod)));

    var blockExpr = Expression.Block(
        new[] { sbVarExpr, iVarExpr, loopVarExpr, rxVarExpr },
        sbNewExpr,
        sbOpenBracketExpr,
        iInitExpr,
        loopExpr,
        sbCloseBracketExpr,
        rxAssignExpr,
        cmdTextAssignExpr);

    return blockExpr;
}
```

And that's it! We now have support for collection parameters. That's a very common use case covered, which'll make querying with Blazer a lot easier. This also means we've pretty much finished up the parameter factory for now, so next time we can start on the data mapper!

Current code state is [on GitHub](https://github.com/b-w/Blazer/tree/a1c75a5e276162285c66893f8644cdf26e0d2547/Blazer).
