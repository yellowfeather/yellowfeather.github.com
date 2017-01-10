---
title: Userland SPI on the BeagleBone with Ubuntu
date: 2012-03-26 00:00:00 Z
categories:
- BeagleBone
layout: post
comments: true
alias: "/2012/03/userland-spi-on-the-beaglebone-with-ubuntu/"
author: chrisr
---

For a project I'm working on I want to get a BeagleBone talking to a <a title="nRF24L01+" href="http://iteadstudio.com/store/index.php?main_page=product_info&amp;cPath=7&amp;products_id=53&amp;zenid=p5ib08mkl698ilq9ch2eqfcf82">nRF24L01+</a> module. The first step is to get userland SPI access working on the BeagleBone, and last week I spent some time getting this going.<a id="more"></a><a id="more-112"></a> The good news is that Robert C Nelson has merged my <a title="Patch to add BeagleBone userspace SPI support" href="https://github.com/RobertCNelson/linux-dev/pull/3">patch</a> into his github repo <a href="https://github.com/RobertCNelson/linux-dev">linux-dev</a> and <strong>a prebuilt image is available to <a href="http://rcn-ee.net/deb/oneiric-armel/v3.2.0-psp6/">download</a></strong>. Read on to learn how to build from source and get details of the patch.<!-- more -->

## Standard Ubuntu running on the BeagleBone
Reading <a title="Userland SPI working on BeagleBone - how best to share?" href="https://groups.google.com/forum/?fromgroups#!topic/beagleboard/GjWljMXyAJM">this</a> post on the BeagleBoard Google Group, led me to Brian Hensley's blog post <a title="SPI working on the Beagleboard XM rev C" href="http://www.brianhensley.net/2012/02/spi-working-on-beagleboard-xm-rev-c.html">SPI working on the Beagleboard XM rev C</a>. With some minor modifications I got Ubuntu installed onto an SD card and running on my BeagleBone; first off download a stable release of Ubuntu for ARM:

{% highlight sh %}
wget http://rcn-ee.net/deb/rootfs/oneiric/ubuntu-11.10-r6-minimal-armel.tar.xz
{% endhighlight %}

And then extract and change directory:
{% highlight sh %}
tar xJf ubuntu-11.10-r6-minimal-armel.tar.xz
cd ubuntu-11.10-r6-minimal-armel
{% endhighlight %}

Then run the script to install the files on the SD card (replace '/dev/sdd' with the drive name for your SD card):
{% highlight sh %}
sudo ./setup_sdcard.sh --mmc /dev/sdd --uboot bone
{% endhighlight %}

Once completed (approx 10 minutes), insert the SD card into your BeagleBone, and all being well, Ubuntu will be up running on your BeagleBone. Verify by running:
{% highlight sh %}
uname -a
{% endhighlight %}

which should give:
{% highlight sh %}
Linux omap 3.2.0-psp6 #2 Thu Mar 22 10:36:44 GMT 2012 armv7l armv7l armv7l GNU/Linux
{% endhighlight %}

and

{% highlight sh %}
lsb_release -a
{% endhighlight %}

should give:
{% highlight sh %}
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 11.10
Release:        11.10
Codename:       oneiric
{% endhighlight %}

<h2>Setup to enable rebuilding the Linux kernel</h2>
In order to enable userland SPI we need to be able to rebuild the kernel, so grab the Linux source:
{% highlight sh %}
git clone git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git
{% endhighlight %}

Install the required tools and libs by running:
{% highlight sh %}
sudo apt-get install gcc-4.4-arm-linux-gnueabi git ccache libncurses5-dev u-boot-tools lzma
{% endhighlight %}

Note that I am running on Debian 6.04 in a VirtualBox VM and had to add the following sources to /etc/apt/sources.list:

{% highlight sh %}
deb http://www.emdebian.org/debian/ squeeze main
deb http://ftp.uk.debian.org/emdebian/toolchains squeeze main
deb http://emdebian.bytesatwork.ch/mirror/toolchains squeeze main
{% endhighlight %}

As per <a href="http://elinux.org/BeagleBoardUbuntu#Demo_Image">these</a> instructions, clone Roberts git repo with:
{% highlight sh %}
git clone git://github.com/RobertCNelson/linux-dev.git
cd linux-dev
{% endhighlight %}

switch to the am33x-v3.2 branch with:
{% highlight sh %}
git checkout origin/am33x-v3.2 -b am33x-v3.2
{% endhighlight %}

Now copy system.sh.sample to system.sh and update system.sh with the settings for your setup:
{% highlight sh %}
cp system.sh.sample system.sh
{% endhighlight %}

For my setup I had to make the following changes to system.sh:

uncomment line 14, change from:
{% highlight sh %}
#CC=arm-linux-gnueabi-
{% endhighlight %}

to:
{% highlight sh %}
CC=arm-linux-gnueabi-
{% endhighlight %}

at line 60 update the path to your linux source, change from:
{% highlight sh %}
#LINUX_GIT=~/linux-stable/
{% endhighlight %}

to:
{% highlight sh %}
LINUX_GIT=~/linux-stable/
{% endhighlight %}

uncomment line 70 to set the kernel entry point, change from:
{% highlight sh %}
#ZRELADDR=0x80008000
{% endhighlight %}

to:
{% highlight sh %}
ZRELADDR=0x80008000
{% endhighlight %}

uncomment line 80 to set the BUILD_UIMAGE flag, change from:
{% highlight sh %}
#BUILD_UIMAGE=1
{% endhighlight %}

to:
{% highlight sh %}
BUILD_UIMAGE=1
{% endhighlight %}

and finally at line 89 uncomment and set the path to the SD card, change from:
{% highlight sh %}
#MMC=/dev/sde
{% endhighlight %}

to:
{% highlight sh %}
MMC=/dev/sdd
{% endhighlight %}

<h2>Build the Linux kernel</h2>
We can now run the buid_kernel.sh script which will clone the Linux source from our linux-stable directory, update to the latest version in git and apply the patches for running on the BeagleBone (this includes the userland SPI support patch :-) ).
{% highlight sh %}
./build_kernel.sh
{% endhighlight %}

Whilst the script is running the Kernel Configuration screen will be shown, to enable SPI scroll down to the 'Device Drivers' option, hit enter and scroll down to 'SPI Support' and hit enter again, now make sure the following are selected:
{% highlight sh %}
[*] Debug support for SPI devices
[*] McSPI driver for OMAP
[*] User mode SPI device driver support
{% endhighlight %}

<h2>Test userland SPI</h2>
Once the build had completed, takes a few hours on my setup, there will be a uImage file in ~/linux-dev/deploy, install this to your SD card with:
{% highlight sh %}
./tools/load_uImage.sh
{% endhighlight %}

Install the SD card into the BeagleBone, and check that SPI is available by running :
{% highlight sh %}
ls /dev/spi*
{% endhighlight %}

on the bone, which should give:
{% highlight sh %}
/dev/spidev2.0
{% endhighlight %}

We can also run a further test by cross compiling spidev_test.c in the VM:

{% highlight sh %}
cd ~/linux-stable
arm-linux-gnueabi-gcc Documentation/spi/spidev_test.c -o spitest
{% endhighlight %}

Remove the SD card from the BeagleBone and transfer spitest to the SD card:

{% highlight sh %}
cp spitest /media/rootfs/home/ubuntu/spitest
{% endhighlight %}

reinsert the SD card into the BeagleBone, connect pins 29 and 30 together on header P9, boot and run:

{% highlight sh %}
sudo ./spitest -D /dev/spidev2.0
{% endhighlight %}

All being well you should see:

{% highlight sh %}
spi mode: 0
bits per word: 8
max speed: 500000 Hz (500 KHz)

FF FF FF FF FF FF
40 00 00 00 00 95
FF FF FF FF FF FF
FF FF FF FF FF FF
FF FF FF FF FF FF
DE AD BE EF BA AD
F0 0D
{% endhighlight %}

Next step is to connect the SPI to an nRF24L01+ module and get them talking!

Many thanks to Robert C Nelson, Branden Hall and Craig Berscheidt.


<h2>Userland SPI support patch details</h2>
FYI: the userland SPI support patch is based on the <a href="https://groups.google.com/forum/#!topic/beagleboard/B3akyoyjwG4/discussion">patch</a> from Craig Berscheidt and makes the following changes to ~/linux-dev/KERNEL/arch/arm/mach-omap2/board-am335xevm.c:

Adds the bone_am335x_slave_info struct:
{% highlight sh %}
static struct spi_board_info bone_am335x_slave_info[] = {
  {
    .modalias      = "spidev",
    .irq           = -1,
    .max_speed_hz  = 12000000,
    .bus_num       = 2,
    .chip_select   = 0,
  },
};
{% endhighlight %}

adds an init function:
{% highlight sh %}
/* setup beaglebone spi1 */
static void bone_spi1_init(int evm_id, int profile)
{
  setup_pin_mux(spi1_pin_mux);
  spi_register_board_info(bone_am335x_slave_info, 
    ARRAY_SIZE(bone_am335x_slave_info));
  return;
}
{% endhighlight %}

and adds the following line to beaglebone_dev_cfg by:
{% highlight sh %}
  {bone_spi1_init,	DEV_ON_BASEBOARD, PROFILE_ALL},
{% endhighlight %}
