---
layout: post
title: VirtualBox Notes
---

A collection of how-to notes on running Debian VMs on MacOS using VirtualBox.

# Installing the Guest Additions

The Guest Additions allow you to change the screen resolution of your VM (so
fullscreen is actually fullscreen) and improves mouse integration.

On Debian, you need 3 packages before you can install the guest additions:

	sudo apt install dkms build-essential linux-headers-amd64

Then from the VirtualBox menu bar, select "Insert Guest Additions CD image..."
from the "Devices" menu. The Guest Additions CD should appear at
`/media/cdrom0`. Do the following:

	cd /media/cdrom0
	sudo sh ./VBoxLinuxAdditions.run

Then restart the VM. Once it restarts and you log in, the screen resolution
should change automatically to fit your screen.

