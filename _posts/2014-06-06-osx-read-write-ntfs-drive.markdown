---
layout: post
title:  "OSX - Reading/Writing NTFS Drive"
date:   2014-06-06 19:43:38 -0400
categories: osx ntfs
logo: ntfs-osx.jpg
---
NTFS drives are common given the number of Windows devices in the technology ecosystem.
However, Mac OSX (at the time of this post) does not have an easy way to read/write
to/from an external NTFS drive. These instructions explain how to work around the
issue of not being able to interact with an NTFS drive on OSX.

### Process

Googling around, I was able to piece together a small workaround that enabled interaction
with the NTFS file system on OSX. Many of the pieces/steps are taken from the sites mentioned
in the Credit section at the end of this post. Note that these steps assume that the OSX
environment already has [HomeBrew](http://brew.sh/) installed.


{% highlight bash %}
$ ruby -e "$(curl -fsSL https://raw.github.com/mxcl/homebrew/go/install)"
$ brew doctor
$ brew remove fuse4x
$ brew install ntfs-3g
$ sudo mv /sbin/mount_ntfs /sbin/mount_ntfs.orig
$ sudo ln -s /usr/local/Cellar/ntfs-3g/2014.2.15/sbin/mount_ntfs /sbin/mount_ntfs
$ brew install osxfuse

# for info only, if curious
$ brew info osxfuse

$ sudo /bin/cp -RfX /usr/local/Cellar/osxfuse/2.6.4/Library/Filesystems/osxfusefs.fs /Library/Filesystems
$ sudo chmod +s /Library/Filesystems/osxfusefs.fs/Support/load_osxfusefs
{% endhighlight %}

Once the above have been performed, detach and re-attach the external NTFS drive. The drive should
appear, and inspecting the "Get Info" option for the drive should show the drive mounted in read/write
mode.

### Credit

Contributions to some of the above were gleaned from:

[Coolest Guides on the Planet - How to Write to a NTFS Drive from OS X Mavericks](http://coolestguidesontheplanet.com/how-to-write-to-a-ntfs-drive-from-os-x-mavericks/)
