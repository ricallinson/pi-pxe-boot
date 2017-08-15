# Unpacking and Packing piCore.gz

Notes on how to unpack and pack a piCore `.gz` filesystem.

## Unpack

The source `9.0.3v7.gz` file is assumed to be in the current working directory;

	mkdir fs
	cd fs
	gunzip -c ../9.0.3v7.gz | sudo cpio -i -d

## Pack

Packing will produce a new `core.gz` in the parent directory;

	cd fs
	# sudo depmod -a -b $(pwd) `uname -r`
	sudo find . | sudo cpio -o -H newc | sudo gzip -2 > ../core.gz
	sudo advdef -z4 ../core.gz
