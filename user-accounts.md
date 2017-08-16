# User Accounts

How to add a user to the system?

## Idea 1

Generate SSH keys on PXE client startup registering them in a central service.

* Change `tc` user password.
* Generate keys with `ssh-keygen`.
* Upload SSH keys and password to trusted central service.

## Idea 2

Generate SSH keys for each PXE client filesystem and store them in a central service.

* Using [httplist](http://wiki.tinycorelinux.net/wiki:netbooting) download `user.tcz`.
* The `user.tcz` package is created only once on demand and contains SSH keys for the `tc` user.
* Installing `user.tcz` adds the SSH keys for the `tc` user account and sets its password.

Subsequent PXE requests from the same MAC address invalidate prior SSH keys.
