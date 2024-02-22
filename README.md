# RootFileSystemIntegrity
## Integrity for Root File System

We need to provide both read and write access for the root file system. I have made the use of the technique of overlay filesystem. An overlay fs allows a read-write filesystem to be overlaid into another read-only filesystem. There are typically two layers, one is the lower layer which is read-only and the other is the upper layer which is writable. All modifications go to the upper writable layer. In this case the lower layer is the read-only dm-verity and the upper layer is the writable layer which is ramfs which is a RAM based filesystem.

In order to implement overlay fs technique we first need to create a dm-verity partition and then mount it. For this the following steps are to be implemented -

**RESIZING DISK PARTITION/ CREATING NEW PARTITIONS**

1. Use `parted -l` to check the partitions available on the disk.
2. Next enter gdisk to open the partition you want to reformat, ex. `gdisk /dev/sda`
3. An interface opens up in which you can query the disk to see the various configurations and memory related information. For example - Use `p` to check the size of each partition.
4. Once the size of the partitions has been determined, we need to delete the root partition and then resize it to make room for creating new partitions for the purpose of overlaying    filesystems. To do that enter `d`. This will delete the current partition.
5. Next use `n` to create a new partition.
6. Enter the previous parition number and press enter for the start block size. Use the default start block available.
7. Next enter the amount of memory you want to allocate to the newly created partition. Ex. +30G
8. After this enter `p` to check if the parition has been properly and successfully been created.
9. Then create the new partitions using the above steps.
10. Once everything looks good and you are ready to save the changes, use `w` to write the changes and to update the GPT table.
11. After the GPT table has been updated, run `partprobe` to inform the kernel about the newly created partitions.
12. To verify that the Linux kernel can see the partition, use command `cat /proc/partitions`.

**MOUNT FILE SYSTEM ON PARTITIONS**

Choose the file system type that you want to assign to your partition, for example, I want the partition to be of ext4 type.

1. Issue the command `mkfs.ext4  /dev/sda1`.
2. Issue the `blkid` command to list all known block storage devices and look for the newly created partition in the output.

   Dont mount it yet
4. To create a mount point, exeucte command - `mkdir /mnt/mount_point_for_dev_sda1`.
5. To mount the filesystem on the newly created mount point, run the command -
   
   `mount -t ext4 /dev/sda1  /mnt/mount_point_for_dev_sda1/`


6. Make use of the `df -h` command shows which filesystem is mounted on which mount point.

**CREATING AND MOUNTING THE DM-VERITY DEVICE**

1. First step is to create a file path which stores the hash verification data of the non root device. The verity device can be a path and it will write the hash tree.

   `veritysetup format <root device> <verity device> | grep Root | cut -f2 >> roothash.txt`
3. Creates a mapping with <name> backed by device <data_device> and using <hash_device> for in-kernel verification.
   
   `veritysetup open <root device> <name><verity device> $(cat roothash.txt)`
4. Now create a mount point and mount the verity device from **/dev/mapper/<name>**

   `mount /dev/mapper/non-root /mnt/dm-verity`

**MOUNTING RAMFS AND OVERLAYING BOTH THE LAYERS**

1. Once the dm-verity device has been mounted, next step is to make the upper layer writable and to do that we use **ramfs** and mount that file system.

   `sudo mount -t ramfs ramfs /ram-dir`

2. Now we have both the layers ready and now we can execute the **overlay fs** command.
  
3. In the mount point where we mounted ramfs we need to create two directories, one upper and work which we will use in the mount overlay command.

   `mount -t overlay overlay -o lowerdir=mnt/dm verity,upperdir=mnt/ramfs_dir/upper,workdir=mnt/ramfs_dir/work mnt/merged`

**INITRD**
unmkinitramfs /boot/initrd.img-$(uname -r) initramfs/








