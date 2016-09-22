---
title: Model binding with Glass Mapper and Sitecore
category: Sitecore
---

When I first started using [Glass Mapper](http://glass.lu) with Sitecore, I had lots of code like this in my controllers.

```csharp
using System.Web.Mvc;
using Glass.Mapper.Sc;
using SAFC.Site.Models;

namespace SAFC.Site.Controllers
{
    public class WidgetController : Controller
    {
        public ViewResult Widget()
        {
            var context = new SitecoreContext();

            var model = context.GetCurrentItem<Widget>();

            return View(model);
        }
    }
}
```

Basically - get a new context... get me the current item mapped to my model type... stuff it in a view... done.
There are a few things that you can do to tidy this up. Injecting the context in a constructor is an obvious example. 

Second attempt
--------------

```csharp
public class WidgetController : Controller
{
    private readonly ISitecoreContext _context;

    public WidgetController(ISitecoreContext context)
    {
        _context = context;
    }

    public ViewResult Widget()
    {
        var model = _context.GetCurrentItem<Widget>();

        return View(model);
    }
}
```

There were a few issues that I was still not happy about though.

1. It doesn't use the datasource for renderings. It just grabs the current item, blindly. 
2. You end up with `ISitecoreContext` all over the place. Every controller now has a dependency on it. 
3. You're still mapping the model in every action. 

A better approach
-----------------

The next approach was to use [model binding](https://docs.asp.net/en/latest/mvc/models/model-binding.html). 

```csharp
public class WidgetController : Controller
{
    public ViewResult Widget([Glass] Widget model)
    {
        return View(model);
    }
}
```

Now we're getting somewhere! No more dependency on `ISitecoreContext`, and we don't need to explicitly bind the model on each action. 

Note how we've decorated the model being passed in with the `Glass` attribute. This is what tells ASP.Net MVC to use our new model binder. 
Here's what the code for `GlassAttribute` looks like.

```csharp
public class GlassAttribute : CustomModelBinderAttribute, IModelBinder
{
    private readonly IGlassBinder _binder;

    public GlassAttribute() : this(DependencyResolver.Current.GetService<IGlassBinder>())
    {
    }

    public GlassAttribute(IGlassBinder binder)
    {
        _binder = binder;
    }

    public bool InferType { get; set; }
    public bool IsLazy { get; set; }

    public virtual object BindModel(ControllerContext controllerContext, ModelBindingContext bindingContext)
    {
        _binder.InferType = InferType;
        _binder.IsLazy = IsLazy;

        return _binder.BindType(bindingContext.ModelType);
    }

    public override IModelBinder GetBinder()
    {
        return this;
    }
}
```

You'll probably recognize the `InferType` and `IsLazy` properties if you've done any work with Glass before. They're used here to be able to pass down our preferences for how we want our mapping to work to the binder. 

Because this is an Attribute, it's not easy to pass in an `IGlassBinder`, so I've done the next best thing and grabbed it from the current `DependencyResolver`. There is still a public constructor which takes an `IGlassBinder` for use in tests, etc.

Speaking of `IGlassBinder`... 

```csharp
public interface IGlassBinder
{
    bool InferType { get; set; }
    bool IsLazy { get; set; }
    object BindType(Type modelType);
    T BindType<T>();
}
```

... and here's the implementation...

```csharp
public class GlassBinder : IGlassBinder
{
    private readonly ISitecoreContext _sitecoreContext;

    public GlassBinder(ISitecoreContext sitecoreContext)
    {
        _sitecoreContext = sitecoreContext;
    }

    public bool InferType { get; set; }
    public bool IsLazy { get; set; }

    public object BindType(Type modelType)
    {
        if (_sitecoreContext == null)
            throw new Exception("Unable to resolve dependency for ISitecoreContext.");

        var item = Sitecore.Mvc.Presentation.RenderingContext.CurrentOrNull?.Rendering?.Item ??
                   Sitecore.Context.Item;

        return _sitecoreContext.CreateType(modelType, item, IsLazy, InferType, null);;
    }

    public T BindType<T>()
    {
        return (T) BindType(typeof (T));
    }
}
```

Phew! There's a few things going on here, so lets break it down. 

First, this is where we've moved our dependency on `ISitecoreContext`. It's a single place, so that's an improvement on shotgunning it all over the controllers. 

```csharp
public GlassBinder(ISitecoreContext sitecoreContext)
{
    _sitecoreContext = sitecoreContext;
}
```

In the `BindType` method, we do a little validation to check that we can get a valid `ISitecoreContext` and throw an exception if not. 

```csharp
if (_sitecoreContext == null)
    throw new Exception("Unable to resolve dependency for ISitecoreContext.");
```

Next we work out what `Item` we need to bind to - has a content editor set a datasource on the rendering, or are we defaulting to the current Sitecore item?

```csharp
var item = Sitecore.Mvc.Presentation.RenderingContext.CurrentOrNull?.Rendering?.Item ?? 
           Sitecore.Context.Item;
``` 

Finally, we map the model using our Glass context, and return it. 

```csharp
return _sitecoreContext.CreateType(modelType, item, IsLazy, InferType, null);
```

Conclusion
----------

I've found this to be a really nice way to work. The mapping of my models is all nicely kept in a single place. It copes well with datasources, and makes my controllers very simple to test. 

I've also extended this to add other attributes like `GlassHome` for mapping the `Home` item for the current site. This is used for things like a footer rendering which stores data in a single place for the whole site. 

```csharp
public ViewResult Footer([GlassHome] NavigationSource source)
{
    return View(source.Footer);
}
```

There is also a `GlassCurrentItem` attribute for when you have some properties that you need from the current item, as well as others that are coming from a datasource. One example for a place where this is useful is if you're trying to get related content for a specific article page. The information about the rendering (background colour, title) can come from the datasource, and the information about the page (tags) can come from the page. 

```csharp
public ActionResult RelatedContent([Glass] RelatedContentModule module, [GlassCurrentItem] ContentPage page)
{
    var model = _processor.Process(new PopulateRelatedContentQuery(module, page));

    return View(model);
}
```

Happy data binding!