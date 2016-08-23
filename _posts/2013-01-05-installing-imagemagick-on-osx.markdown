---
layout: post
title:  "Installing ImageMagick on OSX"
date:   2013-01-05 17:07:38 -0400
categories: imagemagick osx install
---
Instructions related to the installation and configuration of the
[ImageMagick](http://www.imagemagick.org/script/index.php) product on a Mac OSX operating system. The installation
instructions and documentation stem from an initial problem I had related to attemping to install the software
using the Ruby Gem.

## Problem - First Attempt

The first attempt at installing the gem resulted in the following output:

{% highlight bash %}
$ gem install rmagick

No package 'MagickCore' found
checking for stdint.h... yes
checking for sys/types.h... yes
checking for wand/MagickWand.h... no

Can't install RMagick 2.13.1. Can't find MagickWand.h.
*** extconf.rb failed ***
{% endhighlight %}

## Solution

To solve for the above, the following commands were run.

Install software via HomeBrew:

{% highlight bash %}
$ brew install ImageMagick
{% endhighlight %}

Add the package configuration path to the env vars in the .bash_profile file:

{% highlight bash %}
$ vim ~/.bash_profile
# add the following
#   export PKG_CONFIG_PATH="/opt/local/lib/pkgconfig:$PKG_CONFIG_PATH"
{% endhighlight %}

Source the newly-updated .bash_profile file:

{% highlight bash %}
$ source ~/.bash_profile
{% endhighlight %}


Symlink the required libraries and directories:

{% highlight bash %}
$ ln -s /usr/local/include/ImageMagick/wand /usr/local/include/wand
$ ln -s /usr/local/include/ImageMagick/magick /usr/local/include/magick
{% endhighlight %}

At this point, you should have a fully-functioning ImageMagick installation on OSX.
