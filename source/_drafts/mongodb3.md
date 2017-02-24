fdisk /dev/xvde

> Device contains neither a valid DOS partition table, nor Sun, SGI or OSF disklabel
Building a new DOS disklabel with disk identifier 0xca14dbcd.
Changes will remain in memory only, until you decide to write them.
After that, of course, the previous content won't be recoverable.

Warning: invalid flag 0x0000 of partition table 4 will be corrected by w(rite)

Command (m for help): m
Command action
   a   toggle a bootable flag
   b   edit bsd disklabel
   c   toggle the dos compatibility flag
   d   delete a partition
   l   list known partition types
   m   print this menu
   n   add a new partition
   o   create a new empty DOS partition table
   p   print the partition table
   q   quit without saving changes
   s   create a new empty Sun disklabel
   t   change a partition's system id
   u   change display/entry units
   v   verify the partition table
   w   write table to disk and exit
   x   extra functionality (experts only)

Command (m for help): n
Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended
Select (default p): p
Partition number (1-4, default 1): 
Using default value 1
First sector (2048-377487359, default 2048): 
Using default value 2048
Last sector, +sectors or +size{K,M,G} (2048-377487359, default 377487359): 
Using default value 377487359

Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.

> mkfs.ext4 /dev/xvde1

mke2fs 1.42 (29-Nov-2011)
Filesystem label=
OS type: Linux
Block size=4096 (log=2)
Fragment size=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
11796480 inodes, 47185664 blocks
2359283 blocks (5.00%) reserved for the super user
First data block=0
Maximum filesystem blocks=4294967296
1440 block groups
32768 blocks per group, 32768 fragments per group
8192 inodes per group
Superblock backups stored on blocks: 
  32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208, 
  4096000, 7962624, 11239424, 20480000, 23887872

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done  

> mkdir /data
> mount /dev/xvde1 /data

> df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/xvda2       36G  3.0G   31G   9% /
udev            3.9G  4.0K  3.9G   1% /dev
tmpfs           1.6G  748K  1.6G   1% /run
none            5.0M     0  5.0M   0% /run/lock
none            3.7G  4.0K  3.7G   1% /run/shm
/dev/xvde1      178G  188M  168G   1% /data

# 编辑fstab
> UUID=7e9b077b-d488-461c-81cf-4bc18cb58d8c /data         auto    defaults        0       0

> chown -R mongodb:mongodb /data/db
