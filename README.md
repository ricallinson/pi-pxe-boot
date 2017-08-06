# Setup PXE Boot on a Pi

## Creating a Raspbian SD Card

First create a SD card with the latest Raspbian image (these are OSX commands).

	diskutil list
	diskutil unmountDisk /dev/disk<disk# from diskutil>
	sudo dd bs=1m if=2017-07-05-raspbian-jessie-lite.img of=/dev/rdisk<disk# from diskutil> conv=sync

You can press Ctrl+T to see the progress. Once the process has finished boot a Pi with the SD card.

## Enabling USB/Network Boot on a Pi

Boot into a Pi using the SD card created above and log on. At command line execute the following to enable USB booting;

	sudo apt-get update && sudo apt-get upgrade
	echo program_usb_boot_mode=1 | sudo tee -a /boot/config.txt
	sudo reboot

After the reboot login and check the OTP has been programed correctly.

	vcgencmd otp_dump | grep 17:

This should return __17:3020000a__. You can shout down the Pi and remove the SD card.

	sudo shutdown -h now

## USB/Network Boot Image

The SD card you just created can now be used to enable USB/Network booting for each Pi that powers on with it. This machines are now the clients in you cluster.

## Create a PXE Server

Make a new Raspbian SD card or USB drive as in step one. Login and use use `sudo raspi-config` to expand the file system. Then update the system with the following command;

	sudo apt-get update && sudo apt-get upgrade

### Create a Filesystem for the Client 

For the client to boot it will need a filesystem. We will copy the one we're currently using;

	sudo apt-get install rsync
	sudo mkdir -p /nfs/client1
	sudo rsync -xa --progress --exclude /nfs / /nfs/client1

Now we regenerate SSH host keys on the client filesystem by chrooting into it;

	cd /nfs/client1
	sudo mount --bind /dev dev
	sudo mount --bind /sys sys
	sudo mount --bind /proc proc
	sudo chroot .
	rm /etc/ssh/ssh_host_*
	dpkg-reconfigure openssh-server
	exit
	sudo umount dev
	sudo umount sys
	sudo umount proc

### Switch to a Fixed IP Address

To continue you need to know your networks router or gateway IP address, the IP address of this server and the IP address of the networks DNS server;

	ip route | grep default | awk '{print $3}'
	ip -4 addr show dev eth0 | grep inet
	cat /etc/resolv.conf

With the above information noted we now changed the network configuration.

	sudo nano /etc/network/interfaces

Change the line, `iface eth0 inet manual` so that the address is the first address from the command before last, the netmask address as 255.255.255.0 and the gateway address as the number received from the last command.

	auto eth0
	iface eth0 inet static
	    address 10.0.0.88
	    netmask 255.255.255.0
	    gateway 10.0.0.1

Disable the DHCP client daemon and switch to standard Debian networking. Then reboot so the changes take effect;

	sudo systemctl disable dhcpcd
	sudo systemctl enable networking
	sudo reboot

With the networking updated you'll need to manually add a DNS server;

	echo "nameserver 10.0.0.1" | sudo tee -a /etc/resolv.conf

Now make the file immutable so it cannot be changed by other processes;

	sudo chattr +i /etc/resolv.conf

### Install Server Software

Now we install all the services required to create a PXE server.

	sudo apt-get install dnsmasq tcpdump

Stop dnsmasq breaking DNS resolving;
	
	sudo rm /etc/resolvconf/update.d/dnsmasq
	sudo reboot

With `dnsmasq` installed we need to change its configuration so it will respond to DHCP requests from PXE.

	echo | sudo tee /etc/dnsmasq.conf
	sudo nano /etc/dnsmasq.conf

Then replace the contents of `dnsmasq.conf` with;

	port=0
	dhcp-range=10.0.0.255,proxy
	log-dhcp
	enable-tftp
	tftp-root=/tftpboot
	pxe-service=0,"Raspberry Pi Boot"

The value of `dhcp-range` must be the broadcast address of your current network.

Now create a /tftpboot directory and start the `dnsmasq` service.

	sudo mkdir /tftpboot
	sudo chmod 777 /tftpboot
	sudo systemctl enable dnsmasq.service
	sudo systemctl restart dnsmasq.service

With the `dnsmasq` service running tail its logs and then power up a client device on the same physical network;

	tail -F /var/log/daemon.log

At some point you should see something like;

	raspberrypi dnsmasq-tftp[1903]: file /tftpboot/bootcode.bin not found

Stop monitoring by typing `CTRL+C`.

__BUGS:__ If you don't see any requests from the client it could be that you've run into one of the Pi bugs. To see if you have follow these [instructions](https://www.raspberrypi.org/documentation/hardware/raspberrypi/bootmodes/) to use the latest `bootcode.bin`. Now try the `tail` command above again.

Next, you will need to copy `bootcode.bin` and `start.elf` into the `/tftpboot` directory. You should be able to do this by copying the files from `/boot`, since these are the right ones. We need a kernel, so we might as well copy the entire boot directory.

	cp -r /boot/* /tftpboot

Now we restart the `dnsmasq` service so it can serve the boot files;

	sudo systemctl restart dnsmasq

If all has gone well this should now allow the client Pi to boot through until it tries to load a root filesystem (which it doesn't have).

### Serve a Filesystem

For this example we'll use NFS to serve our filesystem.

	sudo apt-get install nfs-kernel-server
	echo "/nfs/client1 *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports
	sudo systemctl enable rpcbind
	sudo systemctl restart rpcbind
	sudo systemctl enable nfs-kernel-server
	sudo systemctl restart nfs-kernel-server

Edit `/tftpboot/cmdline.txt`;

	nano /tftpboot/cmdline.txt

and from root= onwards, and replace it with:

	root=/dev/nfs nfsroot=10.0.0.88:/nfs/client1 rw ip=dhcp rootwait elevator=deadline

You must substitute the IP address here with the IP address of your server.

Finally edit `/nfs/client1/etc/fstab` removing leaving only the line that starts with `proc`;

	nano /nfs/client1/etc/fstab
