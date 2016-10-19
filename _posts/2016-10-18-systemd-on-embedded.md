---
layout: single
categories: [linux, technology, systemd]
---

I run Debian Jessie on my Odroid C1 that I use as a NAS/media server, which means that the controversial [systemd](https://en.wikipedia.org/wiki/Systemd) is used as my init system. I've been using systemd since jessie was in testing and came to like it. It made configuring the boot process much easier and actually taught me a lot more about how Linux internals work. I'd never had a problem with it until powering on my odroid after a few months of it being off.

Normally my Odroid boots up in about 6 seconds and I'm able to SSH in. This time was different, however. I wasn't able to SSH in! Not having any way to view output or debug the boot process, I thought all was lost and I'd have to reinstall! What could have happened after not being powered on for a couple months? I then thought to plug in the external harddrive that I used to store my files. After turning the Odroid back on I was able to log in after about 6 seconds. Why would SSH be dependent on an external harddrive?

So what happened? It turns out that systemd attempts to mount all disks specified in fstab before doing anything else. If a disk cannot be found, it waits up to 90 seconds by default for it to show up. To disable this "feature", simply add the *nofail* option to fstab like so:
```
/dev/sda1       /srv/share1 ntfs nofail 0 0
```
This will make it so that the mount isn't a hard dependency of local-fs.target, which will allow the boot to continue while waiting for the disk.

How does SysV init handle this case? It turns out it just prints an error to the screen and continues happily: 

![Bootup]({{ site.url }}/assets/sysvinit.png)

I still can't decide which behavior is "proper". For an embedded device that is accessed over SSH a boot should probably never hang, so SysV init was right. On the other hand, what if that device was critical for system operation and not having causes major problems? Then systemd would be correct.

Lesson learned: Just when you think you have the hang of something, you find out otherwise. One system is not "more correct" than the other, just different.
