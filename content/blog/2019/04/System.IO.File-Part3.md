---
title: "System.IO.File Part 3"
date: 2019-04-29T21:51:06-05:00
draft: false
---


**System.IO.File Part 3 - Write Locks**

Before I forget, I just want to call out Time Of Check to Time Of Use ([TOCTOU](https://en.wikipedia.org/wiki/Time_of_check_to_time_of_use)). I had not heard of this term until the time of writing this post. That being said, this is exactly the type of condition I will be describing. Additionally, this class of vulnerability is seen way too often and typically under-exploited. These timing attacks or race conditions are usually a result of an accident by a developer and by a researcher, meaning the condition is not intentional or apparent, nor is the discovery. This is why many of these types of bugs are found accidentally. I have been fortunate enough to (accidentally or not) find some accidental bugs. The story behind some of them will have to wait until another time.

The third and hopefully not last installment of playing with System.IO.File has to do with my first post [System.IO.File-Part 1](https://p0shkatz.github.io/blog/2019/01/System.IO.File-Part1). Once I identified the issue and someone to help fix it, I was asked to test the solution to the original problem. The original problem was that files that were executed by a process running as NT AUTHORITY\LOCAL SYSTEM could be written to by any Authenticated User, an obvious Access Control List (ACL) issue. The solution because of other limitations and/or unknown consequences was to replace the original identity (AU) with the CREATOR OWNER principal as part of the new ACL. This solution seemed promising from both a defensive and offensive standpoint. From a defense standpoint, only the creator of a file system object could modify it. From an offense standpoint, Users still have the ability to create objects (wink, wink).

So here was the problem. This privileged service was creating file objects and executing them. I wanted to replace one of them with my custom executable. My previous exploit was not working now because LOCAL SYSTEM is the owner of the file and my FileSystemWatcher cannot beat the new ACL. Instead, I prepopulated the expected file structure and target binary. That makes me the owner. Cool! Wait a sec, prepopulated? The application that I am abusing with this and my Part 1 post is a client application responsible for communicating with a privileged service that does privileged things. As a mature service, everything it does is logged using a custom and verbose logging facility. In order to find a binary that was potentially susceptible to this new condition I suspected was present, I wrote a quick script to parse the logs.

**TIP**: Logging can work against an adversary. Logging can also work against a defender.

So, once the objects are there, I executed the normal process which invoked the privileged service. Unfortunately, the LOCAL SYSTEM account just overwrote my binary. Continuing on down this path, I reviewed the ACL to see how my user principal and my token could fit into this picture. As I mentioned, Users still had the ability to create new file and folder objects in the root directory. The problem is that SYSTEM is overwriting them. My next attempt was to restrict Write or Modify permissions to only my account. Unfortunately how the Users access control entry was configured did not allow me to change permissions on a file.

I had a thought... but had no clue. What if I (as the OWNER) do not allow anyone to touch my stuff? Is that possible, is that a thing that would even prevent SYSTEM from overriding? Using the principal of "exclusive lock" or "write lock", I prepopulated the target binary and did the following:

`$fileStream = [System.IO.File]::Open("C:\path\to\file.ext", [System.IO.FileMode]::Open, [System.IO.FileAccess]::Write, [System.IO.FileShare]::None)`

What does this do? The opens a handle to the target file with write access and declines this or other processes from accessing the file (including read access). This is our exclusive or write lock mechanism. My first assumption is that an exclusive lock would not prevail between an unprivileged process that holds the lock and a privileged process running as LOCAL SYSTEM. I quickly found out, this is not the case. My assumption is that by design, LOCAL SYSTEM will not interfere with user processes that require this type of file management to prevent corruption or to maintain application stability.

This is where things got somewhat problematic. I did not want the installer to overwrite (W/M) my binary, but I did want it to execute (R/E) it. This means that the initial write had to be prevented, but the subsequent request to read and execute the file had to be permitted. One way to do that is monitor for that specific file access request from LOCAL SYSTEM. At this point, I did not have the ability to monitor those calls since I had not escalated my privileges. Long story short, I guessed. I viewed the timing that was stored in the logs of previous executions. Based on that I grabbed the write lock, executed the application, and slept while grepping the log for a string that indicated it was about to attempt the execution of the binary. While monitoring my script as well as running processes, I could tell I missed the window.

I'd like to tell you that I found some precise method of determining the exact time to release the exclusive lock, but I can't. There was a lot of trial and error. Using the log data and slight adjustments to how I was tailing the log, I was able to reproduce it every time. Sorry if the timing thing seems a bit odd. It was odd, but reliable.

Check out my script on the following gist:

https://gist.github.com/p0shkatz/706f28783bd0358386f1770aa9e445e9

**Additional research**

I had not seen this research on file locks until after I used this technique, but I wanted to call out some previous research that covers this topic much more elegantly than I do. Check out the following for a deeper dive into the subject:

@clavoillotte - https://offsec.provadys.com/intro-to-file-operation-abuse-on-Windows.html
