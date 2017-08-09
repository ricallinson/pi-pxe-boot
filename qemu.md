# In Memory Hypervisor


`tftplist` allows you to load `.tcz` extensions via `TFTP`. This can be used to load the `nfs-utils.tcz` from the network.

The boot options allowed are: `tftplist=server:path/list tftplist=server/path/list`.

* `server` is the tftp server name or IP address.
* `path` is the tftp path to the list file.
* `list` is a file that contains a list of tcz extensions to load.

Example: The tftp server is openvz. My tftp server expects all file names to be relative to /tftpboot. The extension to load is `/tftpboot/nfs/mc2/tftp/nfs-utils.tcz`. The list is `/tftpboot/nfs/tftp/tcz`. This file contains:

  nfs/mc2/tftp/nfs-utils.tcz

So my boot option is `tftplist=openvz:/nfs/tftp/tcz.lst`.

If your tftpserver has icmp blocked (can't ping) add `:no-ping` to the end of the option. Ex: `tftplist=openvz:/nfs/tftp/tcz.lst:no-ping`.
