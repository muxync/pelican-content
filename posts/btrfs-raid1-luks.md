Title: Btrfs RAID1 on LUKS encrypted disks
Slug: btrfs-raid1-luks
Date: 2021-04-29 20:53
Modified: 2021-04-29 20:53
Authors: Mark Mulligan
Category: SysAdmin
Tags: btrfs, luks, raid
Summary: Replacing a failed HDD with Btrfs RAID1 SSDs using LUKS disk encryption


# Finally

In [my previous post](firefox-corrupt.html) I mentioned my HDD was starting to fail.  Well, it finally happened.  Luckily I had backups of all my files so now it's just a matter of replacing it with something more robust.  I decided on using Btrfs RAID1 on LUKS encrypted SSDs (Solid State Drives).  This setup will maintain the same level of security of my previous LVM on LUKS encrypted disks setup but with some added benefits.  RAID1 means I can lose 1 out of the 2 SSDs I'm installing and Btrfs makes it simple to add another SSD if I need additional disk space.

# Identify

List current block devices (hint: `sdc` is my root disk and `sdd` is where my backup files are):

    :::shell
    $ sudo lsblk
    NAME                            MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
    sda                               8:0    0 931.5G  0 disk
    sdb                               8:16   0 931.5G  0 disk
    sdc                               8:64   0  55.9G  0 disk
    ├─sdc1                            8:65   0  46.6G  0 part
    │ └─sdc1_crypt                  254:0    0  46.6G  0 crypt /
    └─sdc2                            8:66   0   9.3G  0 part  /boot
    sdd                               8:48   0 931.5G  0 disk
    └─sdd1                            8:49   0 931.5G  0 part
      └─vg_backup-lv_backup         254:2    0 931.5G  0 lvm
        └─vg_backup-lv_backup_crypt 254:3    0 931.5G  0 crypt /backup

Install `hwinfo`:

    :::shell
    sudo apt install hwinfo

Verify the disk model (omit the ` | grep ...` to see all disks):

    :::shell
    $ sudo hwinfo --block --short | sort | grep 'Samsung SSD 860'
      /dev/sda             Samsung SSD 860
      /dev/sdb             Samsung SSD 860

So I know my *new disks* are:

    :::shell
    sda
    sdb

# Wipe

Create a temporary encrypted containers on the partitions:

    :::shell
    sudo cryptsetup open --type plain -d /dev/urandom /dev/sda to_be_wiped_sda
    sudo cryptsetup open --type plain -d /dev/urandom /dev/sdb to_be_wiped_sdb

Verify the temporary encrypted containers exist:

    :::shell
    $ sudo lsblk | egrep -A1 '^sda|^sdb'
    sda                               8:0    0 931.5G  0 disk
    └─to_be_wiped_sda               254:5    0 931.5G  0 crypt
    sdb                               8:16   0 931.5G  0 disk
    └─to_be_wiped_sdb               254:6    0 931.5G  0 crypt

Wipe the containers with zeros (using `if=/dev/urandom` is **not required** as the encryption cipher is used for randomness):

    :::shell
    # If you specify a `bs` in the commands below you can make it go faster
    sudo dd if=/dev/zero of=/dev/mapper/to_be_wiped_sda status=progress
    sudo dd if=/dev/zero of=/dev/mapper/to_be_wiped_sdb status=progress

Close the temporary encrypted containers on the partitions:

    :::shell
    sudo cryptsetup close to_be_wiped_sda
    sudo cryptsetup close to_be_wiped_sdb

# Partition

Create a partition on each disk (required for using a UUID):

    :::shell
    sudo fdisk /dev/sda
    > n
    > Enter
    > Enter
    > Enter
    > Enter
    > p
    > w

    sudo fdisk /dev/sdb
    > n
    > Enter
    > Enter
    > Enter
    > Enter
    > p
    > w

For example:

    :::shell
    $ sudo fdisk /dev/sda

    Welcome to fdisk (util-linux 2.33.1).
    Changes will remain in memory only, until you decide to write them.
    Be careful before using the write command.

    Device does not contain a recognized partition table.
    Created a new DOS disklabel with disk identifier 0xb3bb8ee7.

    Command (m for help): n
    Partition type
       p   primary (0 primary, 0 extended, 4 free)
       e   extended (container for logical partitions)
    Select (default p):

    Using default response p.
    Partition number (1-4, default 1):
    First sector (2048-1953525167, default 2048):
    Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-1953525167, default 1953525167):

    Created a new partition 1 of type 'Linux' and of size 931.5 GiB.

    Command (m for help): p
    Disk /dev/sda: 931.5 GiB, 1000204886016 bytes, 1953525168 sectors
    Disk model: Samsung SSD 860
    Units: sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disklabel type: dos
    Disk identifier: 0xb3bb8ee7

    Device     Boot Start        End    Sectors   Size Id Type
    /dev/sda1        2048 1953525167 1953523120 931.5G 83 Linux

    Command (m for help): w
    The partition table has been altered.
    Calling ioctl() to re-read partition table.
    Syncing disks.

# Encrypt

Encrypt the partitions:

    :::shell
    sudo cryptsetup luksFormat /dev/sda1
    sudo cryptsetup luksFormat /dev/sdb1

Open the encrypted partitions with the password:

    :::shell
    sudo cryptsetup luksOpen /dev/sda1 SamsungSSD860_1
    sudo cryptsetup luksOpen /dev/sdb1 SamsungSSD860_2

Close the encrypted partitions:

    :::shell
    sudo cryptsetup close SamsungSSD860_1
    sudo cryptsetup close SamsungSSD860_2

Add the existing boot key to the encrypted partitions:

    :::shell
    sudo cryptsetup luksAddKey /dev/sda1 /root/.keyfile
    sudo cryptsetup luksAddKey /dev/sdb1 /root/.keyfile

Find the UUIDs of the encrypted partitions:

    :::shell
    export UUID_sda=$(sudo blkid | grep '^/dev/sda' | awk -F '"' '{print $2}'); echo $UUID_sda
    export UUID_sdb=$(sudo blkid | grep '^/dev/sdb' | awk -F '"' '{print $2}'); echo $UUID_sdb

Open the encrypted partitions with the boot key:

    :::shell
    sudo cryptsetup luksOpen /dev/disk/by-uuid/${UUID_sda} SamsungSSD860_1 --key-file=/root/.keyfile
    sudo cryptsetup luksOpen /dev/disk/by-uuid/${UUID_sdb} SamsungSSD860_2 --key-file=/root/.keyfile

Check the status:

    :::shell
    sudo cryptsetup status /dev/mapper/SamsungSSD860_1
    sudo cryptsetup status /dev/mapper/SamsungSSD860_2

# Btrfs

Install `btrfs`:

    :::shell
    sudo apt install btrfs-progs

Make the Btrfs filesystem:

    :::shell
    sudo mkfs.btrfs \
      --label home \
      --metadata raid1 \
      --data raid1 \
      /dev/mapper/SamsungSSD860_1 \
      /dev/mapper/SamsungSSD860_2

Get the UUID for filesystem (will be the same for both disks):

    :::shell
    sudo blkid | grep 'LABEL="home"' | awk -F '"' '{print $4}'

Make a mount point for the filesystem (replace `/home/muxync` with your own path):

    :::shell
    sudo mkdir -p /home/muxync

Add the `/etc/crypttab` entries (replace UUIDs with your own):

    :::shell
    SamsungSSD860_1 UUID=320986f6-38a4-4ed2-868a-948434cbca37 /root/.keyfile luks
    SamsungSSD860_2 UUID=3cbbac00-06f3-462e-b8c3-4ba3e78d31e6 /root/.keyfile luks

Add a `/etc/fstab` entry (replace UUID with your own):

    :::shell
    UUID=f12f6848-9652-4929-8afd-ac3494885656 /home btrfs defaults 0 2


Verify it mounts:

    :::shell
    sudo mount /home

View free disk space with `df`:

    :::shell
    sudo df -h /home

For example:

    :::shell
    $ sudo df -h /home/
    Filesystem                   Size  Used Avail Use% Mounted on
    /dev/mapper/SamsungSSD860_1  932G   17M  931G   1% /home

Or view free disk space with `btrfs`:

    :::shell
    sudo btrfs filesystem df /home

For example:

    :::shell
    $ sudo btrfs filesystem df /home
    Data, RAID1: total=1.00GiB, used=512.00KiB
    System, RAID1: total=8.00MiB, used=16.00KiB
    Metadata, RAID1: total=1.00GiB, used=112.00KiB
    GlobalReserve, single: total=16.00MiB, used=0.00B

Or view the `btrfs` filesystem:

    :::shell
    sudo btrfs filesystem show /home

# Restore

Disable COW (Copy on Write) for with large files that change often (e.g. `/home/muxync/foo`):

    :::shell
    mkdir /home/muxync/foo
    lsattr -d /home/muxync/foo
    chattr -R +C /home/muxync/foo
    lsattr -d /home/muxync/foo

Copy over files with rsync (rerun without `--dry-run` when ready):

    :::shell
    rsync -hrav --dry-run /backup/home/muxync/ /home/muxync/

# Verify

Reboot the system:

    :::shell
    sudo reboot

The new disks should automatically mount after you enter your LUKS passphrase.

# References:

  * [Debian Wiki Btrfs](https://wiki.debian.org/Btrfs)
  * [Arch Wiki Drive Preparation](https://wiki.archlinux.org/index.php/Dm-crypt/Drive_preparation)
  * [Arch Wiki Drive Encryption](https://wiki.archlinux.org/index.php/Dm-crypt/Device_encryption)
  * [BTRFS and RAID1 over LUKS](https://balaskas.gr/btrfs/raid1.html)
  * [LVM on LUKS on RAID1 on Arch Linux](https://jasonwryan.com/blog/2012/02/11/lvm)
  * [Debian Stretch Full Disk Encryption](https://gist.github.com/ppmathis/ccfbfce86484dc61834c1f17568d7b80)
