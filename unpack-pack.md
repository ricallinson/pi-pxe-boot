# Unpacking and Packing piCore.gz

Notes on how to unpack and pack a piCore `.gz` filesystem.

## Unpack

The source `9.0.3v7.gz` file is assumed to be in the current working directory;

	mkdir fs
	cd fs
	gunzip -c ../9.0.3v7.gz | sudo cpio -i -d

## Change

Change the filesystem however you want. Commonly by adding commands to the `/opt/bootlocal.sh` script.

	sudo nano ./opt/bootlocal.sh

To synchronize the time and start the SSH server you'd add;

	ntpdate 10.0.0.1
	/usr/local/etc/init.d/openssh start

## Pack

Packing will produce a new `core.gz` in the parent directory;

	cd fs
	# sudo depmod -a -b $(pwd) `uname -r`
	sudo find . | sudo cpio -o -H newc | sudo gzip -2 > ../core.gz
	sudo advdef -z4 ../core.gz
	sudo chmod 755 ../core.gz

## Using

With a new `core.gz` created replace the old one by backing it up first;

	sudo mv /tftpboot/9.0.3v7.gz /tftpboot/9.0.3v7.gz.$(date +%s)
	sudo mv ../core.gz /tftpboot/9.0.3v7.gz
