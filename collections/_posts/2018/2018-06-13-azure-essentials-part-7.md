---
layout: post
title: "Azure Essentials: getting started with Search Services with SQL Server and ASP.NET MVC Core"
categories: blog
---

In this post about the very basics of Microsoft Azure, I'll show how to provision and use Azure's search-as-a-service offering to index and search contents from an Azure SQL Server database. I'll also show how to talk to the search service from an ASP.NET MVC Core application. I'm assuming you've already got the database for this, as well as an existing MVC app.

## Creating a Search Service

First order of business will be provisioning the Search Service. Open the Search Services service:

![](/assets/img/blog/2018/06/search-services-service.png)

Then, hit "add" to start creating a new Search Service:

![](/assets/img/blog/2018/06/search-services-add.png)

The blade for creating a new Search Service will open:

![](/assets/img/blog/2018/06/search-services-create.png)

Like many Azure services, this is one of those things that'll have a public URL. The other important option here is the pricing tier. I **strongly** suggest carefully reviewing the available options, because the price for even the most basic non-free tier is quite high as compared to what we've seen so far from other Azure services. For development work, the free tier is highly recommended.

When you're ready, hit "create" to have Azure provision the resource for you.

## Creating an SQL view

While Azure is busy creating our Search Service, we have some time to create the view in our database that the Search Service will end up using. There are multiple ways of feeding information into the Search Service, but for today we will focus on going the "SQL view" route. We'll create a simple view like so:

```sql
CREATE VIEW [dbo].[AllPhotos]
AS
SELECT
    U.[TenantId],
    P.[Id] AS 'PhotoId',
    P.[Title],
    P.[CreationDate],
    U.[FirstName],
    U.[LastName]
FROM [dbo].[Photos] AS P
INNER JOIN [dbo].[Users] AS U ON U.[Id] = P.[OwnerId]
```

The specific tables and columns we use don't particularly matter, but this will be our working example for the remainder of this exercise.

## Importing the data

After the new Search Service has been created, we can start importing data into it. Open the new Search Service, and hit "import data" from the overview page:

![](/assets/img/blog/2018/06/search-service-import-data.png)

The data import blade will open. The first step will be connecting to a data source:

![](/assets/img/blog/2018/06/search-service-import-data-connect.png)

We have a number of options here. Since we're going to connect to an Azure SQL Server database, select that one:

![](/assets/img/blog/2018/06/search-service-import-data-source.png)

After this, we can give the data source a name and select the Azure SQL database we'd like to connect to:

![](/assets/img/blog/2018/06/search-service-import-data-database-select.png)

After selecting the database, we'll have to provide the credentials that the Search Service will use. I recommend creating a dedicated SQL user for this, which you can then only grant the minimal permissions that are needed for accessing the data the Search Service needs. After entering the credentials, we can select a table or view to use:

![](/assets/img/blog/2018/06/search-service-import-data-view-select.png)

In this case, we select the AllPhotos view we've created in the previous step. After this, we move on to the next step. We'll skip the Cognitive Search integration, and move on to creating the index:

![](/assets/img/blog/2018/06/search-service-import-data-index.png)

Azure creates a suggested index for us using the columns it has found in our view. We can then customize this index further to fit our needs:

![](/assets/img/blog/2018/06/search-service-import-data-index-customize.png)

The key is the unique key for the index. For each column in our view, we can indicate if it should be:

*   Retrievable: indicates that the contents of this column will be included in the result set that gets returned to the client calling the search endpoint. In our case, we only need the PhotoId column, so we only select that one.
*   Filterable: indicates that we want to use this column to restrict the result set. This is not the same as searching. In our case, we select the TenantId column, because we don't want the users of one tenant searching through the data of other tenants.
*   Sortable: pretty self-explanatory. We'll be sorting by CreationDate, so we select that one.
*   Searchable: indicates that we want to be able to search through these columns.

When we've tweaked everything how we want it, we hit "OK" to create the index.

The last step involves setting up the indexer:

![](/assets/img/blog/2018/06/search-service-import-data-indexer.png)

This determines when and how the data should be indexed by the Search Service.

![](/assets/img/blog/2018/06/search-service-import-data-indexer-create.png)

Since we're indexing a view, we can't use integrated change tracking. This means we'll have to use marker columns to indicate when data has been updated. We'll use the CreationDate column for this, since in our scenario data never changes after upload. This also means we won't need to track deletions.

After this is all set, hit "OK" to create the indexer, and then "OK" again to complete the data import.

Depending on how you've set up your scheduling, the indexer might not run right away. You can always run the indexer manually, by opening it from the Search Service overview page:

![](/assets/img/blog/2018/06/search-service-import-data-indexer-run.png)

Azure will keep you up-to-date on the indexing process through notifications:

![](/assets/img/blog/2018/06/search-service-import-data-notification.png)

## Testing the Search Service

After the index has been built, we can explore the Search Service right from the Azure portal. From the Search Service overview page, select "Search explorer":

![](/assets/img/blog/2018/06/search-service-explorer.png)

From there, you can fire simple search queries against a selected index, and observe the raw results:

![](/assets/img/blog/2018/06/search-service-explorer-test.png)

## Integrating with the Search Service

Now that our Search Service is up and running, it's time to integrate it into our MVC application. The first thing we'll need is an access key. From the overview page of the Search Service, open the "Keys" blade:

![](/assets/img/blog/2018/06/search-service-keys.png)

Here we are shown the admin keys. We don't want those. We want a query key, which is far less privileged and can only be used for querying. Hit "Manage query keys" to view them:

![](/assets/img/blog/2018/06/search-service-query-keys.png)

Azure will have already created a key for us by default. We can use this one, or add a new one, it doesn't really matter. We copy it, and add it to our appsettings.json file, along with the name of our Search Service and the index we'll be querying:

```json
{
    "Search": {
        "ServiceName": "its-me-your-search",
        "IndexName": "its-me-your-index",
        "QueryKey": "XXXXXXXXXXXXXXX"
    }
}
```

Next, it's time to write some code. We'll need the [Microsoft.Azure.Search](https://www.nuget.org/packages/Microsoft.Azure.Search) NuGet package.

We first create a model of our search results. Since our results only contain one column (PhotoId), it'll be a very simple model.

```csharp
namespace EpicMVCApp.Models
{
    using System;

    public class PhotoSearchResult
    {
        public Guid PhotoId { get; set; }
    }
}
```

Next, we create a simple class for interacting with the Search Service:

```csharp
namespace EpicMVCApp.Services
{
    using System;
    using System.Collections.Generic;
    using System.Linq;
    using System.Threading.Tasks;

    using EpicMVCApp.Models;

    using Microsoft.Azure.Search;
    using Microsoft.Azure.Search.Models;
    using Microsoft.Extensions.Configuration;

    public class SearchService
    {
        private readonly IConfiguration _configuration;
        private readonly ISearchIndexClient _searchClient;

        public SearchService(IConfiguration configuration)
        {
            _configuration = configuration ?? throw new ArgumentNullException(nameof(configuration));

            var service = configuration.GetValue<string>("Search:ServiceName");
            var index = configuration.GetValue<string>("Search:IndexName");
            var key = configuration.GetValue<string>("Search:QueryKey");

            _searchClient = new SearchIndexClient(service, index, new SearchCredentials(key));
        }

        public async Task<IEnumerable<PhotoSearchResult>> SearchPhotos(Guid tenantId, string query)
        {
            var searchParams = new SearchParameters
            {
                Filter = $"TenantId eq '{tenantId.ToString()}'",
                OrderBy = new[] { "CreationDate desc" }
            };

            var searchResults = await _searchClient.Documents.SearchAsync<PhotoSearchResult>(query, searchParams);

            return searchResults.Results.Select(x => x.Document);
        }
    }
}
```

This allows us to query for photos against our index given a simple query string. We also filter based on TenantId, and sort the results using the CreationDate field. We disregard the search score that Azure returns, since in this particular scenario we just want to sort the results by upload date, to have the most recent photos appear first.

We register this class as a singleton in our Startup.cs:

```csharp
namespace EpicMVCApp
{
    // usings omitted

    public class Startup
    {
        public IServiceProvider ConfigureServices(IServiceCollection services)
        {
            services.AddSingleton<SearchService>();

            // omitted

            return services.BuildServiceProvider();
        }

        // rest of the class omitted
    }
}
```

We can then use our SearchService class as a dependency from any controller or other service, and add our new photo searching functionality to our MVC app.
