---
title: Feature Toggles with Sitecore
category: Sitecore
header: 
    image: posts/toggles/switches.jpg
excerpt: How to use feature toggling 
---

> "Feature Toggling" is a set of patterns which can help a team to deliver new functionality to users rapidly but safely. 

<cite>Martin Fowler</cite> --- [Feature Toggles](http://martinfowler.com/articles/feature-toggles.html)

If you've not heard of feature toggles before, then I suggest you go ahead and read the Martin Fowler article linked above. Since starting to use Continuous Delivery techniques in my development, I've found feature toggles to be an essential tool. 

1. It allows developers to work on a feature for a long time before releasing it to the public. This can be done **without** having to maintain a long lived branch, and all the merge issues which that creates. 
2. It allows control over which features are available on which environment, and to which users. You can (for example) use feature toggles to do phased deployments to specified cohorts of your user-base.  
3. It makes failed deployments and rollbacks much less stressful. Your code doesn't work? It's just a config change to hide your shame. 

I've written (with help from people like [Martin Georgiev](https://twitter.com/marto83), [Dan Curtis](https://twitter.com/dannylee_c) and various others) a library to help bring this handy technique into the Sitecore world. 

You can [clone it from Github](https://github.com/aqueduct/Aqueduct.Toggles) to see how it works, but here's an introduction to some of the features. 

Introduction
------------
In it's simplest state, you'll have a config file defining your available toggles. 

```xml
<?xml version="1.0" encoding="utf-8" ?>
<featureToggles>
    <feature name="ExampleFeature" enabled="true" />
</featureToggles>
```

In your code, you might have a very simple `if` statement. Take a new code path if the toggle is turned on, otherwise use the existing behaviour.

```csharp
if (FeatureToggles.IsEnabled("ExampleFeature"))
{
    //New behaviour
}
else
{
    //Original behaviour
}
```

So now, we've got some brand new code - committed to master, totally untested, and not ready for users. By adding a [config transform](https://msdn.microsoft.com/en-us/library/dd465318(v=vs.100).aspx) that sets `enabled="false"` in the Production environment, we're still safe to deploy. Hooray! 

**Note:** This is all defined in the Aqueduct.Toggles assembly, so you can use it in any .Net project, not just Sitecore ones.
{: .notice--info}

Feature Toggles in Sitecore
---------------------------
Usually when I'm developing a new feature in Sitecore, it will be a brand new rendering. In this case, you're in luck! There's no need for feature toggles - just don't put the rendering on any pages, and the users won't see it. 

Sometimes however, I've got a more difficult feature to work on. Perhaps I'm replacing an existing rendering with a totally new one, or switching a whole layout? Sometimes these are defined in multiple places, or come from standard values, making them hard to swap out.

The way we did this in Aqueduct.Toggles.Sitecore was to override the processors that select a layout and renderings. We then also extended the toggle config to allow you to add some additional properties to the basic feature definition.  

### Scenario 1 - Swap a rendering for another type of rendering

This is the most basic scenario. You might use this to replace a popular rendering which is used on multiple pages and templates throughout the site, such as navigation.

```xml
<feature name="SwapRenderingsFeature" enabled="true">
    <renderings>
        <rendering name="navigation" 
                   originalId="{AD909EB2-A8E8-484C-BA23-E0CC137142A1}" 
                   newId="{E8A4B6F9-E787-45A1-AB8A-3883405C4436}" />
    </renderings>
</feature>
```

The feature above tells Sitecore that every time it sees a rendering with ID `{AD909EB2-A8E8-484C-BA23-E0CC137142A1}` that you want to swap it with a totally different rendering with ID `{E8A4B6F9-E787-45A1-AB8A-3883405C4436}`. 

### Scenario 2 - Swap the whole layout and renderings for a template type

Another case that I've come across might be that a specific template type (say "News Articles") might be getting a design refresh. In this case, you might want to switch out the layout and renderings coming from standard values, for many thousands of pages at once. 

```xml
<feature name="NewLayoutForTemplateType" enabled="true">
    <templates>
        <template id="{03AFA791-2A92-46E8-8A10-47EC6502B633}" 
                  newLayoutId="{0C993911-CCAB-4303-8D6F-9811E0BB0847}">
            <rendering name="navigation" 
                       placeholder="topnav" 
                       id="{BBDBC750-D502-4B1B-A5B4-77A4AB947DE8}" />
            <rendering name="content" 
                       placeholder="main" 
                       id="{039BF107-3806-464E-B137-CF46A139D1F8}" />
            <rendering name="footer" 
                       placeholder="pagebottom" 
                       id="{E30B60DC-87EF-4B31-8031-B07B3324BDD8}" />
        </template>
    </templates>
</feature>
```

### Scenario 3 - Swap the layout and renderings for a specific item

It's also possible to swap the layout and renderings for a specific item. This might be useful if you want to refresh a homepage, or a specific landing page. 

```xml
<feature name="featurewithsublayouts" enabled="true">
    <items>
        <item id="{9E316C3C-9494-4C99-8AF6-653560D20F76}" 
              newLayoutId="{0C993911-CCAB-4303-8D6F-9811E0BB0847}">
            <rendering name="navigation" 
                       placeholder="topnav" 
                       id="{BBDBC750-D502-4B1B-A5B4-77A4AB947DE8}" />
            <rendering name="content" 
                       placeholder="main" 
                       id="{039BF107-3806-464E-B137-CF46A139D1F8}" />
            <rendering name="footer" 
                       placeholder="pagebottom" 
                       id="{E30B60DC-87EF-4B31-8031-B07B3324BDD8}" />
        </item>    
    </items>
</feature>
```

Overrides
---------

One other useful feature of this library is the concept of overrides. By implementing `IOverrideProvider`, you can provide alternative ways to toggle a feature on or off. 

```csharp
public interface IOverrideProvider
{
    string Name { get; }
    Dictionary<string, bool> GetOverrides();
    void SetOverrides(Dictionary<string, bool> overrides);
}
```

To configure the library to use the new Provider, you need to initialize it. 

```csharp
var provider = new FancyOverrideProvider();
FeatureToggles.SetOverrideProvider(provider);
```

One provider that we've included is the `CookieOverrideProvider`. By setting an encrypted cookie, it is possible to allow a tester to browse the site with a feature enabled, *even if the feature is disabled for everyone else*. 

This is a really popular ability with clients - they love to see new features working with Production content. 

Remember earlier, when I mentioned that you could use feature toggles to enable features for cohorts of users? This is where you'd do it. Create a new `CohortOverrideProvider` and configure it, and then you're ready to do fancy phased rollouts! 
{: .notice--info}

Front end feature toggles
-------------------------

One other place where you might run into difficulty is when you've got CSS or JavaScript changes that need to go along with your new feature. These are all applied client-side, in the browser. How to control these from the server? 

The way we work around this is to add a series of CSS classes to the `<body>` tag of your HTML. Your (simplified) layout file might look like this.

```html
<!doctype html>
<html lang="en">
    <head>
    </head>
    <body class="@FeatureToggles.GetCssClassesForFeatures(Sitecore.Context.Language.Name)">
        @RenderBody()
    </body>
</html>
```

This renders in a browser with a list of all enabled features. 

```html
<body class="feat-EnabledFeature feat-AnotherEnabledFeature no-feat-DisabledFeature">
```

This enables the CSS and JavaScript to target only enabled features.

Continuous Delivery
-------------------

This is just one technique to allow developers to minimise the amount of time spent branching and merging, as to concentrate on shipping code as quickly and painlessly as possible.

The site should always be in a state where you can push to Production at any point, without users getting "incomplete" features and a bad experience. 

