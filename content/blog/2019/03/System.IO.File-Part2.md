---
title: "System.IO.File Part 2"
date: 2019-03-01T06:27:57-07:00 
draft: false
---


**System.IO.File Part 2 - Disk Filler plus more**

This is the second part of my System.IO.File series. This one is even more basic than part 1 (FileSystemWatcher). As pentesters/bug hunters we regularly break things, intentionally or not. Fortunately for me, this one was intentional.

The history on this discovery stemmed from some other work where I needed to test an exfiltration method. The problem was I didn't have any large files to use. Sure, I could have looked around for some non-sensitive, benign files large enough to presumably trigger DLP controls. I had already been playing around with System.IO.File for a couple months. There was one method that I hadn't experimented with and I was curious what it did and how it works.

**SetLength()**

The SetLength method allows you to specify the number of bytes (or size or length) of a specific file stream that the current process has a write handle on. Seems practical. You can specify values such as 100 (for 100 bytes) or 1GB (for 1 Gigabyte). As soon as the SetLength method is called there is an immediate allocation for the file space. What does the space look like? Essentially just null bytes, but the file address table is allocated from disk and associated with the current file... immediately. Cool, I can use this to allocate 20 GB files to use for exfil testing and there's nothing sensitive so I don't need to worry about accidentally sending my personal info. And it's fast.

At this point, I hadn't realized what I might be able to use this method for other than creating some test files. But then I started to wonder what depends on disk space. What would happen if there wasn't any disk space left? So let's try this:

* Get the available disk space from WMI
* Create a test file (acquire handle)
* SetLength to available disk space
* Watch what happens, what breaks

Initially I didn't notice much. Well that's lame. Ok, so let's continuously do this with a loop. Also, WMI is slow, so let's use System.IO.DriveInfo.AvailableFreeSpace:

* Create a test file (acquire handle)
* While true do
* Get the available disk space from DriveInfo
* SetLength to available disk space
* Repeat

Ok, while that's running, now what? Notification from Windows that I'm low an disk space. A couple of user apps don't really open up right. What else? Let's try running this exercise as a logon script. Logging in was sort of painful. Inside the System event log there are a few indications of stuff not being happy. Random stuff. All of these products have a dependency on available disk space. Neat.

What happens if I run this at computer startup? I replace a legitimate service binary with one that executes the above steps. This was a custom Windows service, so I'm not sure at startup what may get executed prior to it (order of execution), but anything that starts after may struggle if the same disk availability requirement exists. After a reboot, my system is totally riding the struggle bus. Open the event viewer and see even more errors than before. This time, some significant products won't even start.

So I can DoS my own box that I obviously have admin on, big deal. Let's go back to user mode and do some additional poking around without privileges. Oops, time for a break. When I come back, I hit CTRL+ALT+DEL to unlock my screen and I notice that not only is it asking for my password, but its also asking for my username. That's odd, but whatever. Next I open Internet Explorer to hit some site that requires IE for "a better user experience". IE opens to the default MSN website. WTF, where's my home page? Hold on a sec, what's happening? I open event viewer once more, check the logs for suspicious errors. Certain Group Policy extensions are failing to apply. Let's fix that, run gpupdate /force. Reopen IE, same thing. At this point, my custom service with my SetLength loop is still running, so I kill that and replace the original binary. Gpupdate /force, reopen IE, and there we go, back to normal.

Certain (not all presumably) Group Policy extensions require disk space to write setting values to disk (registry). The extension for IE is one of those. That's not super interesting because I can literally open regedit and modify those in HKEY_CurrentUser. What about that weird logon thing I saw? That setting is part of the Interactive Logon collection of settings. In fact, that is part of the Security Settings extension which are part of Computer Configuration. I quickly lock and unlock my screen only to find that the condition no longer exists. But wait, lets run the initial script as an unprivileged user to consume all available space, run gpupdae /force as an unprivileged user, and check our lock screen again. Hello missing username! So as unprivileged user, I can takeover the whole system disk and force some privileged computer settings (HKEY_LocalMachine) to get screwed up.

So what's going on here? Using a privileged process, I ran gpresult before and after to try and compare what was changed. I gave up really quickly since there was a ton of settings being configured by Group Policy. Security Settings at least during Group Policy foreground processing (gpupdate **/force**), but possibly background processing (random intervals 90 minutes +/- 30) are all reapplied. What I hadn't realized is that during this processing period, all of the security settings are removed and reapplied. What happens when my script is running is that the settings are removed, my script detects free space and consumes it before the security settings can be reapplied. Even stranger, there is a value configured for some or all of these settings, but its not coming from Group Policy. So where is it coming from?

I decided to reach out to one of my Microsoft contacts for some help in trying to understand the result. I explained the situation and he did some research on what the resulting values might be coming from. Here's what he found:

"On a non-domain joined machine, the SmTblTattoo table is blank. When the machine joins the domain and policy is initially applied, each applied setting (not Preferences) are written to the table (using the value of policy at the time of application). Any values that are not impacted by Group Policy are not written. Then, if the policy does not apply (either by making the policy out of scope, or policy processing failure), any settings in that policy are reverted. Then when policy is applied (either the same policy or a new policy that effects that setting), the policy value takes effect on the machine."

So effectively what I've done is reverted the Group Policy configured security settings to the initial values when the machine was first joined to the domain, which may or may not match what is currently set in Group Policy. I thought was really interesting. We brainstormed a bit on how this might be prevented, but it didn't seem like there was enough concern for the issue for it to be formally addressed. 

Beyond messing with Group Policy, there were definitely some other Microsoft and third party products that did not respond well to the disk availability condition. It's a simple trick with some simple results.

Here's the script I wrote:
https://gist.github.com/p0shkatz/f1b170acd0e9dcab6616dfe5861a8c32