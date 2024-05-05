---
layout: post
title: "Building Blazer, part 1"
categories: blog
---

I was looking at light-weight alternatives to Entity Framework the other day, and it sparked the idea of writing my own simple little data access library. I don't intent to build an entire ORM; rather I like the idea of just making raw ADO.NET a bit easier to use. Normally running a query through ADO.NET requires quite a bit of legwork. For instance, this is a simple query just for fetching a single record from a database:

```csharp
using (var cmd = m_conn.CreateCommand())
{
    cmd.CommandText = "SELECT * FROM [Production].[TransactionHistory] WHERE [TransactionID] = @Id";

    var dbParam = cmd.CreateParameter();
    dbParam.Direction = ParameterDirection.Input;
    dbParam.DbType = DbType.Int32;
    dbParam.ParameterName = "@Id";
    dbParam.Value = 42;
    cmd.Parameters.Add(dbParam);

    using (var reader = cmd.ExecuteReader())
    {
        if (reader.Read())
        {
            var transaction = new TransactionHistory();

            transaction.TransactionID = reader.GetInt32(0);
            transaction.ProductID = reader.GetInt32(1);
            transaction.ReferenceOrderID = reader.GetInt32(2);
            transaction.ReferenceOrderLineID = reader.GetInt32(3);
            transaction.TransactionDate = reader.GetDateTime(4);
            transaction.TransactionType = reader.GetString(5);
            transaction.Quantity = reader.GetInt32(6);
            transaction.ActualCost = reader.GetDecimal(7);
            transaction.ModifiedDate = reader.GetDateTime(8);

            return transaction;
        }
        else
        {
            return null;
        }
    }
}
```

That's a lot of boilerplate for something so basic. I figure a lot (if not most) of it can be automated at runtime using reflection and expressions. That's the direction I'll be going with on this project. I've named it Blazer and its main focus will be writing ADO.NET boilerplate for me. This makes my second objective performance: hand-crafted ADO.NET is about as fast as you can go, and I want to see how close I can get to it.

I'll be writing up my experiences with this project as I go, starting with today's progress.

## Parameter mapping

My first goal will be to do parameter mapping. I'll aim to automate this chunk of code:

```csharp
IDbCommand command = null;
var par = command.CreateParameter();
par.DbType = DbType.Int32;
par.Direction = ParameterDirection.Input;
par.ParameterName = "@Id";
par.Value = 42;
```

I want to accomplish this by generating a parameter factory at runtime. The parameter factory will be a function that can apply input parameters of a certain type to an `IDbCommand`. It'll have the following signature:

```csharp
delegate void ParameterFactory(IDbCommand command, object parameters);
```

When you apply this function to a command, it'll automatically add a parameter to the command for each property in the parameters object.

So, how do we create a parameter factory? Well, we use a parameter factory factory. We'll build the parameter factory using expressions and compile it as a lambda of type `ParameterFactory`.

We'll go line by line. The first step involves calling `IDbCommand.CreateParameter()` on the command variable that will be passed to the factory function. We'll first get the method through reflection:

```csharp
var commandType = typeof(IDbCommand);
var createParamMethod = commandType.GetMethod("CreateParameter", BindingFlags_InstanceProp);
var commandParamExpr = Expression.Parameter(commandType, "command");
```

And call it on the command parameter, assigning the result to a variable:

```csharp
// IDbDataParameter dataParam;
var dbParamVarExpr = Expression.Variable(typeof(IDbDataParameter));

// dataParam = command.CreateParameter();
var createParamExpr = Expression.Assign(
                    dbParamVarExpr,
                    Expression.Call(commandParamExpr, createParamMethod));
```

That gives us the first line.

The next three lines are fairly straightforward: we assign a couple of properties on the data parameter:

```csharp
// dataParam.Direction = ParameterDirection.Input;
var paramDirectionProperty = dataParameterType.GetProperty("Direction", BindingFlags_InstanceProp);
var directionExpr = Expression.Assign(
    Expression.Property(dbParamVarExpr, paramDirectionProperty),
    Expression.Constant(ParameterDirection.Input));

// dataParam.DbType = <some_type>;
var paramDbTypeProperty = dataParameterType.GetProperty("DbType", BindingFlags_InstanceProp);
var dbTypeExpr = Expression.Assign(
    Expression.Property(dbParamVarExpr, paramDbTypeProperty),
    Expression.Constant(DbType.Int32)); // TODO: Type -> DbType mapping

// dataParam.ParameterName = "@<prop_name>";
var paramNameProperty = dataParameterType.GetProperty("ParameterName", BindingFlags_InstanceProp);
var nameExpr = Expression.Assign(
    Expression.Property(dbParamVarExpr, paramNameProperty),
Expression.Constant("@" + prop.Name));
```

(the `prop` variable contains the `PropertyInfo` object for the current property that's being added from the parameter input object)

To assign the parameter value, we need to read it of the input parameters object. The input object is always of type `Object`, so we must first cast it:

```csharp
// var parametersTyped = (<params_type>)parameters;
var parametersType = parameters.GetType();
var typedParamsVarExpr = Expression.Variable(parametersType);
var typedParamsAssignExpr = Expression.Assign(
    typedParamsVarExpr,
    Expression.Convert(parametersParamExpr, parametersType));
factoryBodyExpressions.Add(typedParamsAssignExpr);
```

We can then assign the parameter value:

```csharp
// dataParam.Value = <prop_value>;
var paramValueProperty = dataParameterType.GetProperty("Value", BindingFlags_InstanceProp);
var valueExpr = Expression.Assign(
    Expression.Property(dbParamVarExpr, paramValueProperty),
    Expression.Convert(
        Expression.Property(typedParamsVarExpr, prop),
typeof(object)));
```

Finally, we group the expressions together to get the five-line code block we wanted:

```csharp
// { ... }
var blockExpr = Expression.Block(
    new[] { dbParamVarExpr },
    createParamExpr,
    directionExpr,
    dbTypeExpr,
    nameExpr,
    valueExpr
);
```

Putting it all together, we get the parameter factory factory:

```csharp
public static class ParameterFactoryFactory
{
    private const BindingFlags BindingFlags_InstanceProp = BindingFlags.Instance | BindingFlags.Public;

    public delegate void ParameterFactory(IDbCommand command, object parameters);

    public static ParameterFactory GetFactory(object parameters)
    {
        //IDbCommand command = null;
        //var par = command.CreateParameter();
        //par.DbType = DbType.Int32;
        //par.Direction = ParameterDirection.Input;
        //par.ParameterName = "@Id";
        //par.Value = 42;

        var commandType = typeof(IDbCommand);
        var dataParameterType = typeof(IDataParameter);
        var parametersType = parameters.GetType();

        var createParamMethod = commandType.GetMethod("CreateParameter", BindingFlags_InstanceProp);
        var paramDbTypeProperty = dataParameterType.GetProperty("DbType", BindingFlags_InstanceProp);
        var paramDirectionProperty = dataParameterType.GetProperty("Direction", BindingFlags_InstanceProp);
        var paramNameProperty = dataParameterType.GetProperty("ParameterName", BindingFlags_InstanceProp);
        var paramValueProperty = dataParameterType.GetProperty("Value", BindingFlags_InstanceProp);

        // (command, parameters) =>
        var commandParamExpr = Expression.Parameter(commandType, "command");
        var parametersParamExpr = Expression.Parameter(typeof(object), "parameters");

        var factoryBodyExpressions = new List<Expression>();

        // var parametersTyped = (<params_type>)parameters;
        var typedParamsVarExpr = Expression.Variable(parametersType);
        var typedParamsAssignExpr = Expression.Assign(
            typedParamsVarExpr,
            Expression.Convert(parametersParamExpr, parametersType));
        factoryBodyExpressions.Add(typedParamsAssignExpr);

        foreach (var prop in parametersType.GetProperties(BindingFlags_InstanceProp))
        {
            // IDbDataParameter dataParam;
            var dbParamVarExpr = Expression.Variable(typeof(IDbDataParameter));

            // dataParam = command.CreateParameter();
            var createParamExpr = Expression.Assign(
                dbParamVarExpr,
                Expression.Call(commandParamExpr, createParamMethod));

            // dataParam.Direction = ParameterDirection.Input;
            var directionExpr = Expression.Assign(
                Expression.Property(dbParamVarExpr, paramDirectionProperty),
                Expression.Constant(ParameterDirection.Input));

            // dataParam.DbType = <some_type>;
            var dbTypeExpr = Expression.Assign(
                Expression.Property(dbParamVarExpr, paramDbTypeProperty),
                Expression.Constant(DbType.Int32)); // TODO: Type -> DbType mapping

            // dataParam.ParameterName = "@<prop_name>";
            var nameExpr = Expression.Assign(
                Expression.Property(dbParamVarExpr, paramNameProperty),
                Expression.Constant("@" + prop.Name));

            // dataParam.Value = <prop_value>;
            var valueExpr = Expression.Assign(
                Expression.Property(dbParamVarExpr, paramValueProperty),
                Expression.Convert(
                    Expression.Property(typedParamsVarExpr, prop),
                    typeof(object)));

            // { ... }
            var blockExpr = Expression.Block(
                new[] { dbParamVarExpr },
                createParamExpr,
                directionExpr,
                dbTypeExpr,
                nameExpr,
                valueExpr
                );

            factoryBodyExpressions.Add(blockExpr);
        }
        var lambdaBlockExpr = Expression.Block(
            new[] { typedParamsVarExpr },
            factoryBodyExpressions);
        var lambdaExpr = Expression.Lambda<ParameterFactory>(lambdaBlockExpr, commandParamExpr, parametersParamExpr);
        var lambda = lambdaExpr.Compile();

        return lambda;
    }
}
```

The code as it is right now can be found [on GitHub](https://github.com/b-w/Blazer/tree/986f2dc65d5c7dc591e642578b11b1f733cca97e). I've actually done some refactoring already, but the idea is the same.

Next time, I'll figure out how to map from input parameter value types to `DbType`, which is the remaining Todo in the code above.
