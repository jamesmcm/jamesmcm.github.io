---
layout: post
title: "How to fix high CPU usage by the CacheDelete daemon in OS X Catalina"
titleen: "How to fix high CPU usage by the CacheDelete daemon in OS X Catalina"
titlesp: "How to fix high CPU usage by the CacheDelete daemon in OS X Catalina"
date: 2020-09-20 00:22:42 +0200
comments: true
categories: [General]
meta: false
---

I recently had an ongoing issue with the `deleted` CacheDelete daemon
using huge amounts of CPU time almost constantly. This post covers how I
fixed it and a few tips for OS X recovery and performance improvements.

<!--more-->

Opinions expressed are solely my own and do not express the views or opinions of my employer.

## deleted daemon

The problem I had was that the `deleted` daemon was often taking 90%+ of
CPU usage on my 2017 Macbook Pro, resulting in massive stuttering when trying to use the
system.

Looking up details for the process, I found that [the daemon is
responsible for purging caches](https://apple.stackexchange.com/questions/254810/what-is-the-deleted-daemon-in-macos)
and had caused issues for others. Unfortunately, on Catalina it is not
trivial to disable due to the read-only System volume and the built-in
[System Integrity Protection](https://support.apple.com/en-us/HT204899)
that forbids any changes to system files, even as root.

By looking at the output of `ps` we can see that the binaries are stored
in `/System/Library/PrivateFrameworks/CacheDelete.framework/`, meanwhile
the page linked above mentions that the registered applications with the
daemon are contained in plist files in `/System/Library/CacheDelete/`.

In order to fix this issue, we can simply move all of those plist files.
Then `deleted` will no longer run. Do __not__ remove the `deleted`
binary itself or your system will fail to boot (at login if on a system
with an encrypted disk).

## Removing the plist files

First we need to disable the System Integrity Protection (SIP), this must be
done in recovery mode. To access recovery mode reboot the Macbook and
press Cmd+R whilst it is booting up, the screen should flash white and
take you to a recovery shell with options to reinstall OS X.

In the menu bar at the top, select `Utilities > Terminal` to access the
terminal and then run `csrutil disable`. You should receive a message
that the SIP is now disabled. Reboot the machine.

Log in as usual and open a terminal window. First you must mount the
System volume as writeable:

```bash
$ sudo mount -uw /
```

And then move the CacheDelete plist files:

```bash
$ sudo mv /System/Library/CacheDelete /System/Library/CacheDeleteBackup
```

Now reboot and `deleted` should no longer run.

Note if you also moved
the actual binaries above then the system will no longer boot, this is
easy to fix (assuming you didn't remove them completely) but involves a
few extra steps if your hard disk is encrypted.

### Mounting an encrypted hard disk from recovery

First boot to recover mode and open a terminal as above. Then you need to unlock the encrypted disk.

Use `diskutil apfs list` to find the correct volume (check the capacity
and label), and copy the Logical Volume UUID (it will be something like
`B807C2A0-577F-3DB0-9002-F82B9137696C`).

Then unlock the disk with your disk password (this may be the same as
your user password):

```bash
$ diskutil apfs unlockVolume B807C2A0-577F-3DB0-9002-F82B9137696C
```

Once it is unlocked correctly, you can mount it with diskutil by name,
e.g."

```bash
$ diskutil mount disk2
```

And then find it mounted in `/Volumes/${LABEL}`, there you can make any
necessary fixes and then reboot.

## Other fixes

The above steps fixed the issue I had with CacheDelete which was a
major problem for a few weeks. But I also recommend the following
improvements.

### Disabling Spotlight

Spotlight (the OS X file searcher) indexes all files for rapid
searching, but this indexing can be CPU and Disk IO intensive and run at
inopportune times (you may see this as the `mds_stores` process). Since I never use Spotlight, I just disable it
completely (I only use [ripgrep](https://github.com/BurntSushi/ripgrep)).

```
sudo launchctl unload -w /System/Library/LaunchDaemons/com.apple.metadata.mds.plist
```

Note that you need the SIP disabled to run that.

### Disabling transparency

Disabling transparency helps to reduce CPU usage by the `WindowServer`
process.

Simply go to the accessibility settings and check Reduce Transparency
under Display.


## Summary

I hope this helps anyone else facing the same issue, it was a huge
frustration for weeks. I've written this as much for my own reference
too!

Hopefully one day we'll be able to use Arch Linux at work too.
