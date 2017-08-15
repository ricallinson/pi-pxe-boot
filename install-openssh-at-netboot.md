# SSH Server

Notes on installing OpenSSH (or any piCore package) with netboot.

## Getting Packages

More ARMv7 packages for piCore can be downloaded from [ibiblio.org](http://distro.ibiblio.org/tinycorelinux/8.x/armv7/tcz/).

## Adding Packages

The option `tftplist` allows you to load `.tcz` extensions via `TFTP`. This can be used to load the `openssh.tcz` and `qemu-arm.tcz` from the the `tftp` server.

The boot options allowed are: `tftplist=server:path/list tftplist=server/path/list`.

* `server` is the tftp server name or IP address.
* `path` is the tftp path to the list file.
* `list` is a file that contains a list of .tcz extensions to load.

Example: The tftp server is `10.0.0.1` which expects all file names to be relative to `/tftpboot`. The extension to load is `/tftpboot/tcz/openssh.tcz` and its dependencies. The list is the `/tftpboot/tcz/tcz.lst` file which contains (for example SSH and QEMU);

Create a new directory with a list file in it;

    sudo mkdir /tftpboot/tcz
    sudo nano /tftpboot/tcz/tcz.lst

Add the following to the file and save it;

	/tcz/gcc_libs.tcz
	/tcz/openssl.tcz
    /tcz/openssh.tcz

Now download the packages just listed into the same directory;

	sudo wget -P /tftpboot/tcz/  http://tinycorelinux.net/9.x/armv6/tcz/gcc_libs.tcz
	sudo wget -P /tftpboot/tcz/  http://tinycorelinux.net/9.x/armv6/tcz/openssl.tcz
	sudo wget -P /tftpboot/tcz/  http://tinycorelinux.net/9.x/armv6/tcz/openssh.tcz

Finally alter the boot option in `/tftpboot/cmdline3.txt`;

	sudo nano /tftpboot/cmdline3.txt

Append the following;

    dwc_otg.lpm_enable=0 console=ttyAMA0,115200 root=/dev/ram0 elevator=deadline rootwait quiet nortc loglevel=3 noembed waitusb=1 tftplist=10.0.0.1:/tcz/tcz.lst

If your tftpserver has icmp blocked (can't ping) add `:no-ping` to the end of the option. Ex: `tftplist=10.0.0.1:/tcz/tcz.lst:no-ping`.

## Starting SSH

Hmm...
