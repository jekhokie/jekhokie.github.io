---
layout: post
title:  "Using ovftool for Disk Conversion"
date:   2013-09-12 11:12:09 -0400
categories: ovftool vmware vm disk
---
Simple example on how to use the [VMware OVF Tool](https://www.vmware.com/support/developer/ovf/)
to convert Virtual Machine disk image formats.

### Prerequisites

This example assumes that you have the [VMware OVF Tool](https://www.vmware.com/support/developer/ovf/)
installed/in your current directory.

### Example

Assuming you have the following setup:

{% highlight bash %}
$ ls -l ./Linux64-core2
# Linux64-core2/
# Linux64-core2/Linux64-core2.vmsd
# Linux64-core2/Linux64-core2.vmx
# Linux64-core2/._Linux64-core2.vmxf
# Linux64-core2/Linux64-core2.vmxf
# Linux64-core2/nvram
# Linux64-core2/vdisk-s001.vmdk
# Linux64-core2/vdisk-s002.vmdk
# Linux64-core2/vdisk-s003.vmdk
# Linux64-core2/vdisk-s004.vmdk
# Linux64-core2/vdisk-s005.vmdk
# Linux64-core2/vdisk-s006.vmdk
# Linux64-core2/vdisk-s007.vmdk
# Linux64-core2/vdisk-s008.vmdk
# Linux64-core2/vdisk-s009.vmdk
# Linux64-core2/vdisk-s010.vmdk
# Linux64-core2/vdisk-s011.vmdk
# Linux64-core2/vdisk.vmdk
{% endhighlight %}

To convert the vmdk format disk image to ovf format, perform the following (assuming that the
ovftool binary is in your current directory):

{% highlight bash %}
$ ./ovftool ./Linux64-core2/Linux64-core2.vmx ./Linux64-core2/Linux64-core2.ovf
{% endhighlight %}

This will create a box in the OVF format `/Linux64-core2.ovf` in the specified directory.
