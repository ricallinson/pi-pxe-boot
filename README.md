# Setup PXE Boot on a Pi

## Creating a Raspbian SSD Card

First create a SSD card with the latest [Raspbian Lite](https://www.raspberrypi.org/documentation/installation/installing-images/README.md) image (these are OSX commands).

	diskutil list
	diskutil unmountDisk /dev/disk<disk# from diskutil>
	sudo dd bs=1m if=2017-07-05-raspbian-jessie-lite.img of=/dev/rdisk<disk# from diskutil> conv=sync

You can press Ctrl+T to see the progress.

## Enabling USB/Network Boot on a Pi

Boot into a Pi using the SSD card created above and log on. At the command line execute the following to enable USB booting;

	sudo apt-get update && sudo apt-get upgrade
	echo program_usb_boot_mode=1 | sudo tee -a /boot/config.txt
	sudo reboot

After the reboot login and check the __OTP__ has been programed correctly.

	vcgencmd otp_dump | grep 17:

This should return __17:3020000a__. You can now shutdown the Pi and remove the SSD card. Once this process is done the Pi does not require an SSD card to network boot.

	sudo shutdown -h now

### Enable Again & Again

The SSD card you just created can now be used over and over again to enable USB/Network booting for any Pi that starts-up with it. Once the Pi is shutdown you can remove the card and enable the next Pi. Remember, once this is done the Pi doesn't require an SSD card to network boot. 

## Create a PXE Server

Make a new Raspbian SSD card or USB drive as in the first step. Login and use use `sudo raspi-config` to expand the file system. Then update the system with the following command;

	sudo apt-get update && sudo apt-get upgrade

### Create a Filesystem for the Client 

For the client to boot it will need a filesystem. We will copy the one we're currently using (we do this now before installing any software that the clients would not require);

	sudo apt-get install rsync
	sudo mkdir -p /nfs/client1
	sudo rsync -xa --progress --exclude /nfs / /nfs/client1

Now regenerate SSH host keys on the client filesystem by chrooting into it;

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

To continue you need to know your networks router IP address, the IP address of the current Pi and the IP address of the networks DNS server;

	# Router IP Address
	ip route | grep default | awk '{print $3}'
	# Current IP Address
	ip -4 addr show dev eth0 | grep inet
	# DNS Server IP Address
	cat /etc/resolv.conf

With the above information noted we can now change the network configuration.

	sudo nano /etc/network/interfaces

Replace the line starting with `iface eth0 inet manual` with the following. The `address` is the current IP address (from the middle command), the `netmask` address is 255.255.255.0 and the `gateway` IP address is the router IP address (from the last command).

	auto eth0
	iface eth0 inet static
	    address 10.0.0.88
	    netmask 255.255.255.0
	    gateway 10.0.0.1

Now disable the DHCP client daemon and switch to standard Debian networking so we can use the static IP address we just added. Reboot so the changes take effect;

	sudo systemctl disable dhcpcd
	sudo systemctl enable networking
	sudo reboot

With the networking updated you'll need to manually add a DNS server. This is the same as the gateway IP address you just added;

	echo "nameserver 10.0.0.1" | sudo tee -a /etc/resolv.conf

To make sure this is not changed by other processes we make the file immutable;

	sudo chattr +i /etc/resolv.conf

### Install the Server Software

Now we install all the services required to create a PXE server.

	sudo apt-get install dnsmasq

First we stop `dnsmasq` from breaking normal DNS resolving and reboot again;
	
	sudo rm /etc/resolvconf/update.d/dnsmasq
	sudo reboot

With `dnsmasq` installed we need to change its configuration so it will respond to DHCP requests from our PXE enabled Pi clients.

	echo | sudo tee /etc/dnsmasq.conf
	sudo nano /etc/dnsmasq.conf

With the `dnsmasq.conf` now empty insert the following. The value of `dhcp-range` must be the broadcast address of your current network. This will be the current IP address with the last number changed to __255__.

	port=0
	dhcp-range=10.0.0.255,proxy
	log-dhcp
	enable-tftp
	tftp-root=/tftpboot
	pxe-service=0,"Raspberry Pi Boot"

Now create a `/tftpboot` directory and start the `dnsmasq` service.

	sudo mkdir /tftpboot
	sudo chmod 777 /tftpboot
	sudo systemctl enable dnsmasq.service
	sudo systemctl restart dnsmasq.service

With the `dnsmasq` service running tail its logs and then power up a PXE enabled client Pi on the same physical network;

	tail -F /var/log/daemon.log

At some point you should see something like (it could take up to 60 seconds);

	raspberrypi dnsmasq-tftp[1903]: file /tftpboot/bootcode.bin not found

Stop monitoring by typing `CTRL+C`.

* __BUGS:__ If you don't see any requests from the client it could be that you've run into one of the few Pi bugs effecting network booting. To see if you have, follow these [instructions](https://www.raspberrypi.org/documentation/hardware/raspberrypi/bootmodes/) to use the latest `bootcode.bin`. Now try the `tail` command above again. If this doesn't fix it Google is your friend.

Once you know the client is sending requests you will need to copy `bootcode.bin` and `start.elf` into the `/tftpboot` directory. We can do this by copying the files from `/boot`. Since we also need a kernel we might as well copy the entire boot directory.

	cp -r /boot/* /tftpboot

Now restart the `dnsmasq` service so it knows the boot files are there to serve;

	sudo systemctl restart dnsmasq

If all has gone well this should now allow the client Pi to boot through until it tries to load a root filesystem (which it doesn't have). You can test this by running the tail command again and then powering on your PXE enabled client Pi.

	tail -F /var/log/daemon.log

When a client successfully connects you'll see log lines of the conversation as it attempts to network boot.

### Serve a Filesystem

Now your PXE enabled clients can successfully request a filesystem we need to provide one. For this example we'll use NFS to the serve one we made earlier.

	sudo apt-get install nfs-kernel-server
	echo "/nfs/client1 *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports
	sudo systemctl enable rpcbind
	sudo systemctl restart rpcbind
	sudo systemctl enable nfs-kernel-server
	sudo systemctl restart nfs-kernel-server

Because the filesystem we are serving was a copy of the current one we need to edit `/tftpboot/cmdline.txt` file to remove unnecessary information;

	nano /tftpboot/cmdline.txt

From `root=` onwards replace it with the following substituting the IP address with the IP address of your server;

	root=/dev/nfs nfsroot=10.0.0.88:/nfs/client1 rw ip=dhcp rootwait elevator=deadline

Finally remove any unnecessary information from `/nfs/client1/etc/fstab`. THis means leaving only the line that starts with `proc`;

	nano /nfs/client1/etc/fstab

With all this completed you can now restart your PXE enable client Pi and watch it boot into Raspbian Lite.
