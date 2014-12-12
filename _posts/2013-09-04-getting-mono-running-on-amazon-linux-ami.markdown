---
layout: post
title: "Getting Mono running on an Amazon Linux AMI"
date: 2013-09-04 14:15
comments: true
categories: [Amazon AWS, Mono]
author: chrisr
---

This morning I spent some time getting mono running on an Amazon AWS EC2 instance. The instance is running a standard Amaxon Linux AMI, and the mono packages aren't available in the default repos. After a bit of searching I found a [post](https://forums.aws.amazon.com/message.jspa?messageID=343133#) on the Amazon Web Services Forum that pointed me in the right direction, and is pretty simple once you know how.

First up, you need to modify `/etc/yum.repos.d/epel.repo`. Under the section marked `[epel]`, change `enabled=0` to `enabled=1` e.g.

{% highlight sh %}
[epel]
name=Extra Packages for Enterprise Linux 6 - $basearch
#baseurl=http://download.fedoraproject.org/pub/epel/6/$basearch
mirrorlist=https://mirrors.fedoraproject.org/metalink?repo=epel-6&arch=$basearch
failovermethod=priority
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-6
{% endhighlight %}

Now you can run:

{% highlight sh %}
sudo yum install mono-core
{% endhighlight %}

