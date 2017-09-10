---
layout: post
title: Mac OS X Notes
---

#Symlinking to iCloud Drive

The iCloud Drive is located in your user library at
`~/Library/Mobile Documents/com~apple~CloudDocs`.
Though it may not seem like it in Finder, there is a difference between
"alias" files made using Finder and symlinks made using `ln -s` on
the command line. I recommend always using symlinks; most non-Finder
programs don't understand alias files.

#Making Debian Install USBs

Debian is distributed as CD images. To make a bootable USB out of them,
you'll first need to convert your image to Apple's "Universal Disk Image
Format" (UDIF) using `hdiutil`:

	hdiutil convert debian-x.x.x-amd64-netinst.iso -format UDRW -o debian.img

Then you can use `dd` to copy that image to your thumb drive (in
this case, `/dev/disk2`; make sure you pick the right one and don't
erase your hard drive!):

	dd if=debian.img.dmg of=/dev/disk2 bs=4m

Note that we set a block size of 4m. The default is 512k, which can lead to
a lot of reads. Your install will proceed faster if you choose a larger
block size.

