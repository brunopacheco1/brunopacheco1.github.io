+++ 
date = 2024-10-28T18:49:00+01:00
title = "Installing FreeBSD on a HP 250 G9"
+++

I’ve acquired a Hewlett-Packard 250 15.6-inch G9 laptop with an Intel 12th Gen Core i3-1215U, 8GB of RAM, and a 500GB SSD. Not impressive hardware, I admit, but I will not run a lot of things locally anyway. So, let’s go directly to the point.

I have installed FreeBSD 14.1. Actually, it is not as daunting as it may sound. The installation process is straightforward and easy to understand. The most complex thing would be partitioning, but in my case, I used the entire disk—no need for dual boot—I let the tool do its job alone.

I will not add screenshots of each step, as much more experienced people have thoroughly covered it. Please refer to [this video](https://www.youtube.com/watch?v=bQKaNbarQKI), from [RoboNuggie](https://www.youtube.com/@RoboNuggie).  Approximately 70% of everything I needed to do is explained in that video.

I will list below what was not covered in that video, like playing protected media, graphics, audio, and WiFi configuration.

## Video
This laptop has the Alder Lake-UP3 GT1 [UHD Graphics], which requires `drm-61-kmod`.
```
$ cd /usr/ports/graphics/drm-61-kmod
$ sudo make install clean
$ sudo sysrc kld_list+=i915kms
```

## Windows Manager
I am using Mate; for that, I simply followed the [FreeBSD Handbook](https://docs.freebsd.org/en/books/handbook/desktop/).

## Chromium and CDM
I found a couple of tutorials online, but here I will share simpler steps for supporting protected media playback—Netflix, Spotify, etc.—in Chromium:
- Install Chromium and [Foreign CDM](https://www.freshports.org/www/foreign-cdm):
```
$ sudo pkg install foreign-cdm chromium
$ sudo sysrc linux_enable="YES"
$ sudo service linux start
```
- Download ports and FreeBSD source code to compile `www/linux-widevine-cdm`.
```
$ sudo pkg update
$ sudo pkg install portsnap git
$ sudo portsnap fetch
$ sudo portsnap extract
$ sudo git clone -b releng/14.1 https://git.freebsd.org/src.git /usr/src
```
- Install [Linux Widevine CDM](https://www.freshports.org/www/linux-widevine-cdm/?branch=2023Q4)
```
$ cd /usr/ports/www/linux-widevine-cdm
$ sudo make install clean
```
- Open `chrome://flags` in Chromium and enable `Cdm Storage Database` and `Cdm Storage Database Migration`:
- Close Chromium and open again.
- You can test if DRM is enabled using https://bitmovin.com/demos/drm.

## Printer
I have an HP OfficeJet, and from past Linux experience, I knew about [HPLIP](https://developers.hp.com/hp-linux-imaging-and-printing). Fortunately, it was also available as a binary to be installed with `pkg install hplip`.

## Audio
Audio configuration is also simple. I needed to load a driver and configure hot-swap from speakers to headphones and vice versa whenever a new device was (un)plugged into the audio jack. I followed [this tutorial](https://freebsdfoundation.org/resource/audio-on-freebsd-quick-guide/).

## Wifi
This was the most difficult part of the configuration. My first attempt was loading the driver compatible with my card. Apparently, the chipset was supported, and it worked at first. But as soon as I started consuming videos, it proved unstable. I tried a WiFi micro adapter, but it was quite slow. After a lot of reading on forums and some headaches, I discovered the [Wifibox project](https://github.com/pgj/freebsd-wifibox).

It’s a clever idea of running WiFi drivers inside a Linux guest VM and bridging the connection back to the host machine. The footprint is quite small, but I can use the full potential of my WiFi. I followed [this tutorial](https://xyinn.org/md/freebsd/wifibox).

## Not tested components
I haven’t tested Bluetooth or HDMI audio output.

## Tested platforms:
- Zoom
- Webex
- GeForce Now
- Netflix
- Amazon Primevideo
- Spotify

Well, that is it  with some manual steps, I have a operating system that is fit to my needs.

**See you | Bis geschwënn | Até mais | À bientôt | Ciao**
