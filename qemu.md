# In Memory Hypervisor

## Default Load

`tftplist` allows you to load `.tcz` extensions via `TFTP`. This can be used to load the `nfs-utils.tcz` from the network.

The boot options allowed are: `tftplist=server:path/list tftplist=server/path/list`.

* `server` is the tftp server name or IP address.
* `path` is the tftp path to the list file.
* `list` is a file that contains a list of tcz extensions to load.

Example: The tftp server is `10.0.0.1`. My tftp server expects all file names to be relative to /tftpboot. The extension to load is `/tftpboot/tcz/nfs-utils.tcz`. The list is `/tftpboot/tcz/tcz.lst` file which contains (for example SSH);

    tcz/openssh.tcz

So my boot option is `tftplist=openvz:/tcz/tcz.lst`. In the default piCore `.img` the file `cmdline.txt` is changed to;

    dwc_otg.lpm_enable=0 console=ttyAMA0,115200 root=/dev/ram0 elevator=deadline rootwait quiet nortc loglevel=3 noembed waitusb=1 tftplist=10.0.0.1:/tcz/tcz.lst

If your tftpserver has icmp blocked (can't ping) add `:no-ping` to the end of the option. Ex: `tftplist=10.0.0.1:/tcz/tcz.lst:no-ping`.

## Getting Packages

All the ARM7 packages for piCore can be downloaded from [ibiblio.org](http://distro.ibiblio.org/tinycorelinux/8.x/x86/tcz/).
