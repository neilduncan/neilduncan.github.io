---
title: Green padlocks for local development
category: SSL
header:
    image: posts/ssl/locks.jpg
excerpt: We've all done it. You've been developing a site for weeks or months, deploy it to the production server, and everything falls over because you've done something dumb related to SSL and browser security. It's easily done. 
---

We've all done it. 

You've been developing a site for weeks or months, deploy it to the production server, and everything falls over because you've done something dumb related to SSL and browser security. It's easily done. 

Usually, this is because either developers don't use SSL at all when building (*really bad*), or are using self signed certificates and are used to having to ignore certificate errors (*not quite as bad, but still bad*).

The way I mitigate this is - when I'm building a site that requires SSL on the server, I _run totally valid SSL certificates on my local machine_. I usually work for [digital agencies](http://www.aqueduct.co.uk), so I run LOTS of different sites on my developer PC. 

What I really want is a self signed wildcard certificate that validates correctly and gives me nice green padlocks in my browser. Here's how to do it.

Install a fake trusted root authority
-------------------------------------
Before we generate our fancy new certificate, we'll need a fake authority to sign it for us. 
On Windows 7, you can run `makecert`, which is a tool that comes with Visual Studio or the Windows SDK. 

```
makecert.exe -n "CN=Ghostwheel Development Root CA,O=Ghostwheel,OU=Development,L=London,S=London,C=UK" -pe -ss Root -sr LocalMachine -sky exchange -m 120 -a sha1 -len 2048 -r
```

If you're on Windows 10, you'll need to run the Powershell command [New-SelfSignedCertificate](https://technet.microsoft.com/library/hh848633). 

Install a wildcard certificate for *.local.com
----------------------------------------------
Next, we need a wildcard certificate, signed by the new root authority. 

```
makecert.exe -n "CN=*.local.com" -pe -ss My -sr LocalMachine -sky exchange -m 120 -in "Ghostwheel Development Root CA" -is Root -ir LocalMachine -a sha1 -eku 1.3.6.1.5.5.7.3.1
```

Bind the certificate to port 443
--------------------------------
To use the new certificate in IIS, it needs to be bound to port 443. This will allow us to have _lots_ of sites all running valid SSL.

Open up powershell then

```posh
dir Cert:\LocalMachine\My
```

Find the thumbprint of the new cert (XXXXXXXXX)

```posh
Import-Module WebAdministration
cd IIS:\SslBindings
Remove-Item .\0.0.0.0!443
Get-Item cert:\LocalMachine\My\XXXXXXXXX | New-Item 0.0.0.0!443
```

Once the steps above have been done once, there's no need to ever do them again. 

Add the binding for your site
-----------------------------
First, add an entry in your [hosts file](https://support.rackspace.com/how-to/modify-your-hosts-file/) for the website you want to use. It needs to be of the form *.local.com for the certificate to validate properly.

```
127.0.0.1 mysite.local.com
```
Finally, we need to add a binding in IIS so that we can serve our local site as `https://`
The IIS UI is a bit deficient here. It's expecting that you'll never have more than one site running against a single certificate (port/IP address). Luckily, we can do it via the command line.

```
%systemroot%\system32\inetsrv\appcmd set site /site.name: mysite.local.com /+bindings.[protocol='https',bindingInformation='*:443:mysite.local.com']
```