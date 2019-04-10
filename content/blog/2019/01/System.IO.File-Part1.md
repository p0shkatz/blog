---
title: "System.IO.File Part 1"
date: 2019-01-20T19:21:54-07:00
draft: false
---

**System.IO.File Part 1 - The FileSystemWatchers**

This is the first blog in a series where I'm going to discuss my favorite .NET class of 2018. I learned some neat tricks/techniques with the class that I thought I'd share some of my findings. None of these are overly complex. Some of these are so fundamental, their value may not be totally apparent, until used in the right context.

For a while now, I have been intrigued with the simple capabilities of System.IO.File. It invokes some of the basic input/output Windows APIs used to read and write data. As you can guess, this is used heavily by .NET developers. I'm going to veer slightly (SQUIRREL!) before we get back to File.

A few years ago when I was working as a blue teamer helping product owners secure their platform and their products, we needed to figure out was overwriting a configuration file. We assumed a legitimate process was doing it, but the vendor could not explain what was causing the behavior. This was my introduction to the (System.IO) FileSystemWatcher class. Using FileSystemWatcher and file auditing, we were able to isolate the offending process (and user) and add some automation to replace the modification with our original values. Think of it as a lightweight version of a file integrity monitoring product.

So how does it work? FileSystemWatcher works using specific event triggers. From Microsoft (https://docs.microsoft.com/en-us/dotnet/api/system.io.filesystemwatcher), "Listens to the file system change notifications and raises events when a directory, or file in a directory, changes." Specifically, a listener can be configured to monitor when files and/or directories are created, changed, deleted, renamed, or on file system errors. FileSystemWatchers can be targeted or filtered. They also support monitoring recursive files and directories. This helped us identify the bug in the product and the vendor quickly responded with a patch.

Flash forward a few more years. I was pentesting an application that was invoked by a userland client over a named pipe (more on named pipes later). The named pipe server would receive parameters from the client, request additional details from a web service, and then write out commands to a batch file on the local file system. The batch file was then executed by the service running the named pipe server. The service was running as LOCAL SYSTEM. The batch file had an access control entry that allowed Authenticated Users to Modify the file. You know where this is going.

The problem was timing. Insert Indiana Jones meme. By the time the file was written, it was executed in a matter of milliseconds. I could try replacing the legitimate file manually, but there would likely be lots of fail. A better idea would be to automate the whole thing, plus I like writing PowerShell. In my opinion, any C# app worth writing is worth PoCing in PowerShell first.

Once I figured out the logic of the service, I wrote a FileSystemWatcher script in PowerShell that works like this:

* Detect the file system OnCreate and OnChange events on target file
* Replace the target file with evil file
* Continue to monitor that evil file exists on target
* Profit

The longer version of my logic is that it would wait for the service to write the file and then replace it. During file creation operations, a process will first open a handle to a target file. At this point in my FileSystemWatcher, the OnCreated event has triggered. Once all or at least partial (asynchronous) bytes have been written to the target file, the act of committing the data to disk triggers the OnChanged event. Fair warning, WriteFile (NtWriteFile) will typically invoke multiple change events.

Next, the script grabs the MD5 hash of the legitimate batch file and stores that for comparison later. Then, the PowerShell script copies over my evil batch file, replacing the legitimate file. It compares the current batch MD5 with the legitimate MD5. If the current MD5 does not match our evil MD5, it continues to copy it (loop/brute-copy). The purpose of checking the MD5 is in case the service hasn't finished commiting changes (sloppy brute-copy). At this point, the service runs my code. I didn't know what to have it execute, so I simply had it run a reverse PowerShell shell.

Eventing in Windows is a powerful concept. File system events are only one part of it. As always, if you want to learn more, I recommend reading up on the official Microsoft documentation. More to come on System.IO.File. Meanwhile, here's the poc I used:

https://gist.github.com/p0shkatz/f81656b7b6ac7ee62fd5ffb77c501133
