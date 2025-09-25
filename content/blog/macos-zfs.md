+++
title = "macOS OpenZFS"
date = "2025-09-24T18:11:21-05:00"
tags = ["fun","macos", "OpenZFS"]
+++

I happened to have a bunch of spare SSDs with me while also running out of space on my M.2 enclosure. So I had the idea of building a quad ssd enclosure DAS.

Most of the enclosures I can find uses the same chipset from ASMedia (ASM-2464PDX), I picked the 4M2 from OWC as it has built-in cooling fans and cheaper than Terramaster.

My specs: 

- OWC 4M2 thunderbolt M.2 bay
- Corsair MP600 PRO LPX 4TB x 4

Since macOS does not have native support for RAID 5 and I don't want to pay for OWC's softRAID, I decided to try [OpenZFS](https://OpenZFSonosx.org/).

Some history on ZFS and macOS ([src](https://OpenZFS.org/wiki/History)): 

- 2009 – Apple's ZFS project closed. The MacZFS project continued to develop the code.
- 2013 – OpenZFS on OS X ports ZFS on Linux to OS X.
- 2013 – Official announcement of the OpenZFS project.



## Getting OpenZFS

Head to [OpenZFS's GitHub release](https://github.com/OpenZFSonosx/OpenZFS-fork/releases) and download the .pkg that supports your current OS. For example: [
OpenZFSonOsX-2.3.0-Sequoia-15-arm64.pkg](https://github.com/OpenZFSonosx/OpenZFS-fork/releases/download/zfs-macOS-2.3.0/OpenZFSonOsX-2.3.0-Sequoia-15-arm64.pkg)

It's gonna to install some daemons and a kernel extension so you will have to restart your mac. It's nice I don't need to worry about disabling SIP.

## Configurations

### Pool/Dataset Creation

1. format all the drives:
	
	you don't have to do this if it's freshly installed
	```sh
	diskutil eraseDisk free none /dev/diskX
	```

2. create `zpool`:

	Following the recommended pool creation command line [here](https://OpenZFSonosx.org/wiki/Zpool)
	assuming my disks are disk4 - disk7:

	```sh
	sudo zpool create -f -o ashift=12 \
	-O compression=lz4 \
	-O casesensitivity=insensitive \
	-O atime=off \
	-O normalization=formD \ 
	[pool] raidz1 /dev/disk4 /dev/disk5 /dev/disk6 /dev/disk7
	```

3. creating dataset (or see below for encrypted dataset):

	```sh
	zfs create [dataset]
	```

4. change `recordsize` for datasets with big files, for example:

	```sh
	sudo zfs set recordsize=1M mypool/photos
	```

5. enable autotrim:

	```sh
	sudo zpool set autotrim=on [pool]
	```

6. mount/unmount dataset:

	```sh
	sudo zfs unmount [dataset]
	```

### Dataset Encryption

1. enable encryption for the zpool:

	```sh
	sudo zpool set feature@encryption=enabled [pool]
	```

2. creating encrypted dataset

	```sh
	sudo zfs create -o encryption=on -o keylocation=prompt -o keyformat=passphrase [dataset]
	```
	
	
3. to see the encryption status of the dataset:

	```sh
	sudo zfs get encryption [dataset]
	```

4. to (un)load the key (so you get to enter it again next time before mounting):

	```sh
	sudo zfs load-key -r [dataset]
	sudo zfs unload-key -r [dataset]
	```
	
	and to check key status:
	
	```sh
	sudo zfs get keystatus [dataset]
	```

## Maintenance

### NO password on read/write

[src](https://OpenZFSonosx.org/wiki/Creating_user_privileges)

by default OpenZFS volumes can only be written by the `root` user, to give yourself read and write access so you don't have to type your password every time:

- Click at the icon of your mounted ZFS pool on your desktop, select in the menu bar File > Get Info.

- Click at the little lock icon at the bottom, type in an admin username and password.

- Click at the Plus icon at the bottom on the left.

- Add a your own user name or all administrators with read and write privileges.

### MISC

- verify checksum and repair with: `sudo zpool scrub [pool]`
- redundancy: `sudo zfs set copies=2 [dataset]`
- current read/write status: `zpool iostat -v [pool]`