# Using piCore for Netboot

## Download piCore

Log on to the Pi and then download the latest [ipCore](http://tinycorelinux.net/9.x/armv6/releases/RPi/) zip file and unzip it;

	wget http://tinycorelinux.net/9.x/armv6/releases/RPi/piCore-9.0.3.zip
	unzip ./piCore-9.0.3.zip

Now mount the `piCore-9.0.3.img` file we just download and check its contents.

	mkdir ./picore_1
	sudo mount -o loop,offset=$((512*8192)) piCore-9.0.3.img ./picore_1
	ls -la ./picore_1

You should see a bunch of files which can now be copied to the `tftpboot` directory. First we'll back up the existing `tftpboot` and make a new one;

## Make piCore the Default Netboot OS

	sudo mv /tftpboot /_tftpboot
	sudo mkdir /tftpboot

Now copy the new files over;

	sudo rsync -xa --progress ./picore_1/ /tftpboot
	ls -la /tftpboot/

You should now see the same list of files as before but in the `tftpboot` directory.

## Try it Out

If all the above was successful power up a PXE enabled Pi on the same physical network and it should now boot into piCore.
