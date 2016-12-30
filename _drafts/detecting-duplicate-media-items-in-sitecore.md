---
title: Detecting duplicate media items in Sitecore
category: Sitecore
header:
    image: posts/duplicates/photos.jpg
excerpt: Automated detection of duplicate items in the Sitecore Media Library
---

Once a Sitecore site has been live for some time, you'll usually see the various databases growing larger and larger. Your backups will take longer, your DBAs will be complaining...

Quite often the biggest culprit is the Media Library. It's time to clean it up.

Generating a hash
-----------------

The approach I've taken to detect duplicate files is to generate a hash on save. This is configured in the `App_Config\Include\Sitecore.Doppelganger.config` file.

```xml
<configuration xmlns:patch="http://www.sitecore.net/xmlconfig/">
  <sitecore>
    <events>
      <event name="item:saved">
        <handler type="Sitecore.Doppelganger.GenerateHash, Sitecore.Doppelganger" method="OnItemSaved"  />
      </event>
    </events>
  </sitecore>
</configuration>
```

It adds a new handler on the `item:saved` event, which computes these hashes, and saves them on the media item.

The handler has various validation code and things to update the item being saved, but the important line is this one.

```csharp
item["hash"] = HashGenerator.GetHash(item);
```

`HashGenerator` is a static class which generates an MD5 hash from a media item's content.

```csharp
using (var md5 = MD5.Create())
using (var content = media.GetMediaStream())
{
    var hash = md5.ComputeHash(content);
    return BitConverter.ToString(hash);
}
```

MD5 is a decent algorithm to pick, as the probability of collisions is [pretty low](http://stackoverflow.com/a/288519/283552). It's also acceptably fast - in the region of 3-5 seconds for a 200mb file on my laptop.
