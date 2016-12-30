---
title: How to enable queries in Sitecore Rendering Data Sources
category: Sitecore
header:
    image: posts/datasources/map.jpg
excerpt: Allow the use of queries in Rendering Data Sources.
---

I recently needed to use a query in a Sitecore Rendering Datasource. I wanted to have a rendering added to the standard values for a page, which used the parent of the page for its data source.

I tried a few different ways before realizing that this is not something that is provided by default!

So. Time to pull out my trusty decompiler, and work out how to add this.

Running the query
-----------------
Sitecore uses an `ItemLocator` to resolve the target item for a data source, so the first thing we need to do is to create a custom one of those.

```csharp
using System.Linq;
using Sitecore;
using Sitecore.Data.Items;
using Sitecore.Mvc.Data;

namespace MyAssembly
{
    public class QueryItemLocator : ItemLocator
    {
        public override Item GetItem(string pathOrId)
        {
            return GetItemByQuery(pathOrId) ?? base.GetItem(pathOrId);
        }

        public override Item GetItem(string pathOrId, Item contextItem)
        {
            return GetItemByQuery(pathOrId) ?? base.GetItem(pathOrId, contextItem);
        }

        public override Item GetItem(string pathOrId, string basePath)
        {
            return GetItemByQuery(pathOrId) ?? base.GetItem(pathOrId, basePath);
        }

        private Item GetItemByQuery(string query)
        {
            if (!query.StartsWith("query:")) return null;

            var innerQuery = query.Replace("query:", "");
            return Context.Item.Axes.SelectSingleItem(innerQuery);
        }
    }
}
```

This will look for data sources starting with the string `query:` and then use the standard Sitecore querying API to find an item. If one isn't found (or if the data source doesn't start with `query:`) then it will fall back to the base behaviour.

Using the new ItemLocator
-------------------------
Next, we need to tell Sitecore to use our new `ItemLocator`. I've done this using a Hook, which is the recommended way to add initialization code.

```csharp
using Sitecore.Events.Hooks;
using Sitecore.Mvc.Configuration;
using Sitecore.Mvc.Data;

namespace MyAssembly.Pipelines
{
    public class QueryItemLocatorInitializer : IHook
    {
        public void Initialize()
        {
            MvcSettings.RegisterObject<ItemLocator>(() => new QueryItemLocator());
        }
    }
}
```

The `MvcSettings.RegisterObject` call will replace the existing registration with our new `ItemLocator`.

To configure this hook to run on application startup, you'll need to add a config patch to your `App_Config\Include` folder.

```xml
<?xml version="1.0" encoding="utf-8" ?>
<configuration xmlns:patch="http://www.sitecore.net/xmlconfig/">
  <sitecore>
    <hooks>
      <hook type="MyAssembly.Pipelines.QueryItemLocatorInitializer, MyAssembly" />
    </hooks>
  </sitecore>
</configuration>
```

Adding to a rendering
---------------------
Now we're ready to use it.

![Datasource with a query](/images/posts/datasources/query-datasource.png "This query refers to the parent of the current item.")

This query refers to the parent of the current item.

Other methods
-------------
I'm not the first to run into this issue, and it looks like there have been a few other attempts to do the same thing.

For example, [this one](http://sitecorepro.blogspot.co.uk/2015/05/sitecore-query-in-mvc-rendering.html) where Konstantin Cherkasov is using a `RenderRenderingProcessor` pipeline.
