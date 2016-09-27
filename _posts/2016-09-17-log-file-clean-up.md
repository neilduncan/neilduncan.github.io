---
title: Log file cleanup
category: SysAdmin
header:
    image: posts/logging/servers.jpg
---

I recently got a call from our sysadmin, asking me to tidy up some of our web servers. My diligent logging had filled up 100s of Gb of disk space, and it was becoming an issue. 


![Log all the things](/images/posts/logging/log-all-the-things.png "Maybe I've gone overboard here?")

I _hate_ having to remember to manually clean stuff up like this, so I put on my Powershell hat, and wrote some scripts to do the job. 
They are designed to delete files older than a certain time, and can be run on specified lists of folders. The scripts are open source, and live [here](https://github.com/aqueduct/Aqueduct.Server.Scripts).

Public Health Warning!
----------------------
<div class="notice--danger">
<p>These will spider a list of folders, and <em>all sub folders</em>, deleting any file that is older than 7 days. 
If you run it on <code>C:\</code> or <code>C:\Users\Neil\non-backed-up-photos-of-my-kids\</code> or something, then you <em>will lose your files</em>.</p>
<p><strong>Please be careful!</strong></p>
</div>

Usage
-----
First clone the repository to the server that you want to keep maintained.

```
git clone https://github.com/aqueduct/Aqueduct.Server.Scripts.git
```

Create a new file in the same directory as the scripts called "folders.txt". Each line in this file is a folder that will be tidied up. Some examples of folders you might want to clean up - IIS logs, Sitecore logs, Temporary ASP.Net files. Here's an example.

```
C:\inetpub\logs\LogFiles
C:\Octopus\Applications\Production\*REDACTED*.Site\Data\logs
C:\Windows\Microsoft.NET\Framework64\v4.0.30319\Temporary ASP.NET Files\root
```

Next, make sure you've allowed Powershell scripts to run. Open a Powershell prompt, and type

```posh
Set-ExecutionPolicy Unrestricted
```

Then you can run the scripts, and tidy up the listed folders. 

```posh	
.\Execute.ps1
```

Scheduling
----------
Of course, if you have to run this manually, it's not much good. Much better to set this up as a scheduled task!

Open up Task Scheduler (`Run > Task Scheduler`), and create a new Task.
Give it a name like "Server Maintenance", and then click the "Change User or Group" button.

![New Task](/images/posts/logging/new-task.png "Change User or Group")

I usually run this task as "SYSTEM".

Next, you'll need to trigger your task. I usually run this weekly at a time when I know that the server is not going to be under load. 

![New Task Trigger](/images/posts/logging/new-task-schedule.png "Weekly, Sunday, 2AM")

Now, we need to tell the scheduler what action to run. The location should be the folder path that the scripts are on.

![New Task Action](/images/posts/logging/new-task-action.png "Powershell .\Execute.ps1")

How it works
------------
The script is super simple. The bulk of the work is done by a function called `Remove-FilesOlderThan` in the `Maintenance-Helpers.psm1` file.

There are a few parameters that you can pass into this function.

```posh     
Param(
    [Parameter(Mandatory=$true)]
    [String] 
    $folderPath,        
    [Int] 
    $days = 30,        
    [Switch] 
    $recurse
)
```

The only mandatory one is `$folderPath` - the rest have sensible default values.

The script does a `Get-ChildItem`, and then loops over the files looking for ones that are older than the specified date. 

```posh  
$age = ($now - $item.CreationTime).Days 
```

If it finds one, it tries to delete, skipping it if it doesn't have access. 

```posh  
try
{
    Remove-Item $item.FullName -Force -ErrorAction Stop
    $deleted += 1
}
catch 
{
	$failed += 1
}
```

Most of the rest of the script is error-handling, and progress bar management, which I won't cover here. 
