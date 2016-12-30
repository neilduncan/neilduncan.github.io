---
title: Add custom information to WFFM emails
category: Sitecore
header:
    image: posts/wffm/emails.jpg
excerpt: Add custom information to emails from Sitecore Web Forms for Marketers.
---
I was recently asked to add some additional non-form based information to an email sent by Web Forms for Marketers. Specifically, they wanted to include a referral code that was stored in the user's session.

After some false starts, I came up with this.

The form
--------
First I set up the form. It had the usual number of fields for the user to fill in.

![Form fields](/images/posts/wffm/form-fields.png "Note there's no 'Referral' field.")

Then I added a Save Action to send an email.
![Save actions](/images/posts/wffm/save-actions.png)

![Save actions](/images/posts/wffm/send-email-action.png)

Here's what I ended up with for the email body text. Note that there's a `[Referral]` token in there, despite there not being a `Referral` field on the form.

![Email body](/images/posts/wffm/email-body.png "Note the [Referral] token.")

Adding the custom data
----------------------
I chose to do this using a ProcessMessage pipeline, which is one of the ones provided by WFFM.

First I added a new processor.

```csharp
using Sitecore.WFFM.Abstractions.Mail;

namespace MyAssembly
{
    public class ProcessMessage
    {
        public void AddCustomData(ProcessMessageArgs args)
        {
            var sidValue =  HttpContext.Current.Session["some-key"] ?? "";
            args.Mail.Replace("[Referral]", sidValue.ToString());
        }
    }
}
```

It is configured using a patch config file in the `App_Config\Include` folder like this. Note that it's patched in _before_ the `SendEmail` processor.

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

Hey presto. Done!
