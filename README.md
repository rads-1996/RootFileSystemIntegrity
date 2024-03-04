# RootFileSystemIntegrity
## Integrity for Root File System

We need to provide both read and write access for the root file system. I have made the use of the technique of overlay filesystem. An overlay fs allows a read-write filesystem to be overlaid into another read-only filesystem. There are typically two layers, one is the lower layer which is read-only and the other is the upper layer which is writable. All modifications go to the upper writable layer. In this case the lower layer is the read-only dm-verity and the upper layer is the writable layer which is ramfs which is a RAM based filesystem.

In order to implement overlay fs technique we first need to create a dm-verity partition and then mount it. For this the following steps are to be implemented -

**RESIZING DISK PARTITION/ CREATING NEW PARTITIONS**
1. Use a USB and connect it with the VM which runs the Ubuntu image.
2. Install a bootloader image software such as "Balena Etcher" on the VM.
3. Download the corresponding ISO image for the Live Ubuntu system.
4. Using the on screen instruction in the Balena software, create a live bootable image.
5. Reboot the main Ubuntu system and enter the grub menu and choose to boot from the live system.
6. Install gparted on the live system if it doesn't already exist.
7. Open gparted and choose the disk of the actual system.
8. Since those partitions are no longer mounted we can make changes to it and resize the root partition.
9. Once the resizing has been done, reboot to the original system and the changes should be visible now.
10. Now new partitions can be created via the gdisk interface as well.
11. To verify that the Linux kernel can see the partition, use command `cat /proc/partitions`.

**MOUNT FILE SYSTEM ON PARTITIONS**

Choose the file system type that you want to assign to your partition, for example, I want the partition to be of ext4 type.

1. Issue the command `mkfs.ext4  /dev/sda1`.
2. Issue the `blkid` command to list all known block storage devices and look for the newly created partition in the output.

3. Make use of the `df -h` command shows which filesystem is mounted on which mount point.

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
1. In order to find the intercept points for inserting the scripts for verity setup and overlayfs mounting, contents of the initrd image are extracted using - **unmkinitramfs**            `unmkinitramfs /boot/initrd.img-$(uname -r) initramfs/`
2. This command will extract the contents of the image and contains an init script which is the driver script responsible for setting up the device and finally mounting the root file system.

