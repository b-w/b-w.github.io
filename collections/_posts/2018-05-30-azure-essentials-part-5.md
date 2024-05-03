---
layout: post
title: "Azure Essentials: getting started with Table Storage"
categories: blog
---

In this post about the very basics of Microsoft Azure, I'll show how to get started with Table Storage. The Table Storage is a key/value store for non-relational structured data. It offers especially high performance on inserts, and is a good option to use for logging.

We'll use the Storage Account from our [previous post](/Blog/Post/107), so we won't have to deal with setting one of those up again.

## Writing to Table Storage

The first thing we'll set up is the ability to write records to the Table Storage in our Storage Account. As a working example, we'll use auditing logs that keep track of logins to our MVC application. We'll first model the log record that we're going to store in Table Storage:

```csharp
namespace EpicMVCApp.Tables
{
    using System;

    using Microsoft.WindowsAzure.Storage.Table;

    public class SecurityAuditLog : TableEntity
    {
        public Guid? UserId { get; set; }

        public string ClientIP { get; set; }

        public bool IsSuccessful { get; set; }

        public string InfoMessage { get; set; }

        public string ExceptionMessage { get; set; }
    }
}
```

It inherits from TableEntity.

Next, we'll create a simple service class for writing to Table Storage:

```csharp
namespace EpicMVCApp.Services
{
    using System;
    using System.Collections.Generic;
    using System.Threading.Tasks;

    using EpicMVCApp.Tables;

    using Microsoft.Extensions.Configuration;
    using Microsoft.WindowsAzure.Storage;
    using Microsoft.WindowsAzure.Storage.Table;

    public class TableStorageService
    {
        private readonly IConfiguration _configuration;

        public TableStorageService(IConfiguration configuration)
        {
            _configuration = configuration ?? throw new ArgumentNullException(nameof(configuration));
        }

        public async Task Insert(string tableName, ITableEntity entity)
        {
            var table = GetTable(tableName);

            await table.CreateIfNotExistsAsync();

            var operation = TableOperation.Insert(entity);

            await table.ExecuteAsync(operation);
        }

        protected CloudTable GetTable(string tableName)
        {
            var storageAccount = CloudStorageAccount.Parse(_configuration.GetConnectionString("MyAzureStorage"));
            var tableClient = storageAccount.CreateCloudTableClient();
            return tableClient.GetTableReference(tableName);
        }
    }
}
```

The Insert method is generic: it'll take any TableEntity and write it to the given table. Of course we could type this to only accept SecurityAuditLog objects, but this'll do for now.

We'll register it in our Startup.cs:

```csharp
namespace EpicMVCApp
{
    // usings omitted

    public class Startup
    {
        public IServiceProvider ConfigureServices(IServiceCollection services)
        {
            services.AddSingleton<TableService>();

            // omitted

            return services.BuildServiceProvider();
        }

        // rest of the class omitted
    }
}
```

...and that it! We can use this class to write to our Table Storage account, for example:

```csharp
await tableService.Insert("SecurityAudit", new SecurityAuditLog
{
    PartitionKey = tenantId.ToString(),
    RowKey = Guid.NewGuid().ToString(),
    UserId = userId,
    IsSuccessful = true,
    InfoMessage = "login successful",
    ClientIP = context.HttpContext.Connection.RemoteIpAddress.ToString()
});
```

The PartitionKey and RowKey fields are inherited from TableEntity. The partition key is important in determining the performance of the table when it starts to scale. We'll just use the Id of the tenant for now. As a general rule of thumb, it's better to have a key that creates multiple smaller partitions, rather than one that just creates a few large partitions. Multiple smaller partitions make it easier for Azure to manage the distribution of the partitions over storage nodes.

## Exploring the data

You can't explore the data in the tables from the Azure portal, but you can use [Storage Explorer](https://azure.microsoft.com/en-us/features/storage-explorer/) for this instead:

![](/assets/img/blog/2018/05/table-storage-explorer.png)

## Retrieving the data

Programmatically fetching the data from our MVC application isn't as trivial as uploading it, but it's close. The main catch is that certain handy APIs aren't available in .NET Core, which I'm using here. The main problem I ran into was that [CloudTable.CreateQuery](https://docs.microsoft.com/en-us/dotnet/api/microsoft.windowsazure.storage.table.cloudtable.createquery?view=azure-dotnet) isn't available. So that's LINQ right out the window, unfortunately. Instead, we'll have to use [ExecuteQuerySegmentedAsync](https://docs.microsoft.com/en-us/dotnet/api/microsoft.azure.cosmosdb.table.cloudtable.executequerysegmentedasync?view=azure-dotnet) like so:

```csharp
namespace EpicMVCApp.Services
{
    // usings omitted

    public class TableStorageService
    {
        // omitted

        public async Task<IEnumerable<SecurityAuditLog>> GetSecurityLogs(string tableName, Guid tenantId)
        {
            var table = GetTable(tableName);
            var query = new TableQuery<SecurityAuditLog>()
                .Where(TableQuery.GenerateFilterCondition("PartitionKey", QueryComparisons.Equal, tenantId.ToString()));

            TableContinuationToken token = null;
            var logs = new List<SecurityAuditLog>();
            do
            {
                var results = await table.ExecuteQuerySegmentedAsync(query, token);

                logs.AddRange(results.Results);
                token = results.ContinuationToken;
            } while (token != null);

            return logs;
        }

        // omitted
    }
}
```

That's it for now on storing structured data in _the cloud_.
