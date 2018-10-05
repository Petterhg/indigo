---
title: "Notes on ZFS with Fedora"
layout: post
date: 2017-12-01
image: /assets/images/markdown.jpg
headerImage: false
tag:
- zfs
- fedora
- dkms
star: true
category: blog
author: petter
description: Getting ZFS up and running again after kernel upgrade
---
So, just upgraded from Fedora 26 to 27 on one of our servers. Fedora has done this very easy now a days by adding
a plugin to the default package manager called `dnf-plugin-system-upgrade`. In this case I had a ZFS pool mounted 
on the server. Something one might expect to still be there after a system upgrade.

{% highlight bash %}   
$ zfs list
The ZFS modules are not loaded.
Try running '/sbin/modprobe zfs' as root to load them.
{% endhighlight %}

{% highlight bash %}   
$ sudo /sbin/modprobe zfs
modprobe: FATAL: Module zfs not found in directory /lib/modules/4.14.13-300.fc27.x86_64
{% endhighlight %}

Doh. Both surprising and not. ZFS always needs to be matched with the kernel version running, so with a new kernel, 
an update of ZFS is always necessary. This goes for all kernel modules. BUT this should be taken care of automatically
by dkms. However, taking a look at the currently active modules on the server I found that there is not a single 
one running.  
Dark times.

{% highlight bash %}   
$ dkms status
nothing
{% endhighlight %}

Whether or not this is the product of the “new” dnf plugin or not I don’t really have the energy to find out.
So, I started with removing the lingering ZFS package for Fedora 26 and then downloaded and installed the latest
and greatest.

{% highlight bash %}   
$ dnf remove zfs-release-1-5.fc26.noarch
$ dnf install http://download.zfsonlinux.org/fedora/zfs-release$(rpm -E %dist).noarch.rpm
$ dnf install kernel-devel zfs
{% endhighlight %}

And then to the punch line – manually install the kernel modules. _Yay_.

{% highlight bash %}   
$ dkms add -m spl -v 0.7.5
$ dkms install -m spl -v 0.7.5
$ dkms add -m zfs -v 0.7.5
$ dkms install -m zfs -v 0.7.5
{% endhighlight %}

The version specified here is the newly installed ZFS package version. Just to make it extra super clear to dkms 
that it’s time to stop fucking around.  
Last step: __REBOOT__
