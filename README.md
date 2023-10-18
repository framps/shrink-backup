# shrink-backup is a very fast utility for backing up your SBC:s into minimal bootable img files for easy restore with autoexpansion at boot

_I made this script because I wanted a universal method of backing up my SBC:s into small img files as fast as possible (with rsync), indepentent of what os is in use._

Autoexpansion tested on **Raspberry Pi** os (bookworm and older), **Armbian**, **Manjaro-arm** and **ArchLinuxARM** for rpi with **ext4** root partition.

**Latest release:** [shrink-backup.v0.9.3](https://github.com/UnconnectedBedna/shrink-backup/releases/download/v0.9.3/shrink-backup.v0.9.3.tar.gz)<br>
[**Testing branch**](https://github.com/UnconnectedBedna/shrink-backup/tree/testing) if you want to have the absolute latest version. Resizing of existing img file and btrfs cloning are next on the roadmap and is being developed here.

**Very fast restore because of minimal size of img file.**

**Can back up any device as long as root is `ext4`**<br>
Default device that will be backed up is detected by scanning what disk-device `root` resides on.<br>
This means that _if_ `boot` is a partition, that partition must be on the **same device as `root`**.<br>
Backing up/restoring, to/from: usb-stick `/dev/sdX` with Raspberry pi os has been tested and works. Ie, writing an sd-card img to a usb-stick and vice versa works.

**Ultra-fast incremental backups to existing img files.** 

See [wiki](https://github.com/UnconnectedBedna/shrink-backup/wiki) for a bit more information about usage.<br>
[Ideas and feedback](https://github.com/UnconnectedBedna/shrink-backup/discussions) is always appreciated, whether it's positive or negative. Please just keep it civil. :)

**Don't forget to make the script executable.**

**To restore a backup, simply "burn" the img file to a device using your favorite method.**<br>
When booting up a restored image with autoresize active, wait until the the reboot sequence has occured. The login prompt may very well become visible before the autoresize function has rebooted.

## Usage
```
sudo shrink-backup -h
Script for creating an .img file and subsequently keeing it updated (-U), autoexpansion is enabled by default
Directory where .img file is created is automatically excluded in backup
########################################################################
Usage: sudo shrink-backup [-Uatyelh] imagefile.img [extra space (MB)]
  -U         Update the img file (rsync to existing backup .img), no resizing, -a is disregarded
  -a         Let resize2fs decide minimum space (extra space is ignored), disabled if using -U
  -t         Use exclude.txt in same folder as script to set excluded directories
             One directory per line: "/dir" or "/dir/*" to only exclude contents
  -y         Disable prompts in script
  -e         DO NOT expand filesystem when image is booted
  -l         Write debug messages in logfile shrink-backup.log in same directory as script
  -h --help  Show this help snippet
########################################################################
Example: sudo shrink-backup -a /path/to/backup.img
Example: sudo shrink-backup -e -y /path/to/backup.img 1000
Example: sudo shrink-backup -Ut /path/to/backup.img
```

The folder where the img file is created will ALWAYS be excluded in the backup.<br>
If `-t` option is selected, `exclude.txt` **MUST exist** (but can be empty) within the **directory where the script is located** or the script will exit with an error.

Use one directory per line in `exclude.txt`.<br>
`/directory/*` = create directory but exclude content.<br>
`/directory` = exclude the directory completely.

If `-t` is **NOT** selected the following folders will be excluded:
```
/lost+found
/proc/*
/sys/*
/dev/*
/tmp/*
/run/*
/mnt/*
/media/*
/var/swap
```

**Rsync WILL cross filesystem boundries, so make sure you exclude external drives unless you want them included in the backup.**

Use `-l` to write debug info into `shrink-backup.log` file located in the same directory as the script.

**Applications used in the script:**
- fdisk (sfdisk)
- dd
- parted
- e2fsck
- truncate
- mkfs.ext4
- rsync

## Info

Theoretically the script should work on any device as long as root filesystem is `ext4`. But IMHO is best applied on ARM hardware.<br>
Since the script uses `lsblk` to figure out where the root resides it does not matter what device it is on.<br>
Even if you forget to disable autoexpansion on a non supported system, the backup will not fail. :)

See [wiki](https://github.com/UnconnectedBedna/shrink-backup/wiki) for a bit more information.<br>
[Feedback](https://github.com/UnconnectedBedna/shrink-backup/discussions) is highly apreciated!<br>

### Order of operations - Image creation:
1. Uses `lsblk` to figure out the correct disk device to back up.
2. Reads the block sizes of the partitions.
3. Uses `dd` to create the boot part of the system + a few megabytes to include the filesystem on root. (this _can_ be a partition)
4. Removes and recreates the `root` partition, the size depends on options used when starting the script.
5. Creates the `root` filesystem with the same `UUID` and `LABEL` as the system you are backing up from. (_MUST_ be `ext4`)
6. Uses `rsync` to sync both partitions. (if more than one)

Added space is added on top of `df` reported "used space", not the size of the partition. Added space is in MB, so if you want to add 1GB, add 1024.

The script can be instructed to set the img size by requesting recomended minimum size from `e2fsck` by using the `-a` option.<br>
This is not the absolute smallest size you can achieve but is the "safest" way to create a "smallest possible" img file.<br>
If you do not increase the size of the filesystem you are backing up from too much, you can most likely keep it updated with the update function (`-U`) of the script.

### Smallest possible image

To get the absolute smallest img file possible, do NOT set `-a` option and set "extra space" to 0

Example: `sudo shrink-backup /path/to/backup.img 0`

This will instruct the script to get the used space from `df` and adding 128MB "*wiggle room*".<br>
If you are like me, doing a lot of testing, rewriting the sd-card multiple times. The extra time it takes each time will add up pretty fast.

Example:
```
-rw-r--r-- 1 root root 3.7G Jul 22 21:27 test.img # file created with -a
-rw-r--r-- 1 root root 3.3G Jul 22 22:37 test0.img # file created with 0
```

**Disclaimer:**<br>
Because of how filesystems work, `df` is never a true representation of what will actually fit in a created img file.<br>
Each file, no matter the size, will take up one block of the filesystem, so if you have a LOT of very small files (running docker f.ex) the "0 added space method" might fail during rsync. Increase the 0 a little bit and retry.<br>
This also means you have VERY little free space on the img file after creation.<br>
If the filesystem you back up from increases in size, an update (`-U`) of the img file might fail.

### Order of operations - Image update:
1. Probes the img file for information about partitions.
2. Mounts `root` partition with an offset for the loop.
3. Checks if multiple partitions exists. If true, reads `fstab` on img file and mounts boot partition accordingly with an offset.
4. Uses `rsync` to sync both partitions. (if more than one)

To update an existing img file simply use the `-U` option and the path to the img file.<br>
Example: `sudo shrink-backup -U /path/to/backup.img`

Changing size in an update is not possible at the moment but is in the todo list for the future.

**Thank you for using my software <3**

*"A backup is not really a backup until it has been restored"*
