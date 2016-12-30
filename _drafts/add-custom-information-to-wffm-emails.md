---
title: Add custom information to WFFM emails
category: Sitecore
header:
    image: posts/ssl/locks.jpg
excerpt: Add custom information to emails from Sitecore Web Forms for Marketers.
---

![Email body](/images/posts/wffm/email-body.png "Note the [Referral] token.")

![Form fields](/images/posts/wffm/form-fields.png "Note there's no 'Referral' field.")

![Save actions](/images/posts/wffm/save-actions.png)

![Save actions](/images/posts/wffm/send-email-action.png)

```csharp
using Sitecore.WFFM.Abstractions.Mail;

namespace MyAssembly
{
    public class ProcessMessage
    {
        public void AddCustomData(ProcessMessageArgs args)
        {
            var sidValue = HttpContext.Current.Session["some-key"] ?? "";
            args.Mail.Replace("[Referral]", sidValue.ToString());
        }
    }
}
```



```xml
<?xml version="1.0" encoding="utf-8" ?>
<configuration xmlns:patch="http://www.sitecore.net/xmlconfig/">
  <sitecore>
    <pipelines>
      <processMessage>
        <processor patch:before="processor[@method='SendEmail']" type="MyAssembly.ProcessMessage, MyAssembly" method="AddCustomData" />
      </processMessage>
    </pipelines>
  </sitecore>
</configuration>
```
