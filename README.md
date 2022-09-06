# How to make voidlinux (or any other linux) RAM accelerated
## RAM accelerated meaning ?
According to alpine linux wiki, [data disk mode](https://wiki.alpinelinux.org/wiki/Alpine_Install:_from_a_disc_to_PC_Engines_APU#Data_Disk_Mode) is a mode that runs from system RAM, thus it enjoys the same accelerated operation speed as "[diskless](https://wiki.alpinelinux.org/wiki/Alpine_Install:_from_a_disc_to_PC_Engines_APU#Diskless_Mode)" mode. (It also uses swap and a mounted /var but that is irrelavelnt)
> This mode is useful for having RAM accelerated servers with variable amounts of user-data that exceed the available RAM size.

In other words, a RAM accelerated server (or desktop if you are going to run desktop linux on it) is a system that has the operating system run from the system RAM *with* or *without* having other drives mounted.

This guide will walk you through how to setup your linux distribution to make it RAM accelerated.

### Advantages
* Read/Write speeds faster than the fastest SSD even if you are booting from a USB1.0
* Can work with literally nothing attached (useful if you have a loose usb slot that can cause troubles)
* Sdcards will not wear out over time since most of the reads/writes are done in RAM. And you are free to save changes all at once when needed (if needed)

### Disadvantages
* Space constrains: Your entire distro needs to fit in RAM
* Boot time: At every boot, your entire system will be copied to RAM which might take time on heavier systems
* Ram usage: Something like firefox needs 300mb of disk space (which is RAM in this case) it also saves a lot of files in ~/.mozilla and ~/.cache which will take up some RAM as well, and firefox (when running) will take more RAM on its own.
PS: I am using this on a computer with 8Gb of RAM. And it is working great.

## Setting things up
### Preparing the medium
* Start buy installing a distribution onto your sdcard/flashdrive.
* In voidlinux (and a lot of other distributions) , the installation medium have an option to boot entirely from RAM. This is useful when you need to install linux onto the same medium.
* Proceed with a normal installation until you have a bootable medium
  * It is recommended not to update your system until after editin the init script of the initramfs.
* Install a text editor and `mkinitcpio`
  * Note: There is nothing stopping you from using `dracut` instead of mkinitcpio
* It might be wise to check the output of `lsblk` to see where partitions should be mounted

### Making the system RAM accelerated
* Edit the file `/lib/initcpio/init` line 44
```ash
# Mount root as /new_root
"$mount_handler" /new_root
```
Replace that line with the following lines.
```ash
# Mounts /new_root as tmpfs and copies root to it
mkdir -p /disk_root /new_root
mount -o size=8G -t tmpfs tmpfs /new_root
"$mount_handler" /disk_root
echo "Your system is being copied to RAM"
cp -a /disk_root/* /new_root
umount /disk_root
```
Here I am using 8G, but you can use as much as you like as long as it is more than the size of your filesystem (and less than you RAM, unless you are using swap)
A copy of the result `init` file is available [here](/init)

* Update your initramfs
  * This should be as simple as running `mkinitcpio -g /boot/initramfs-5.18.19_1.img` or whatever initramfs image you have in your `/boot`. 
  * A better way to do this is to update the `linux` package as kernel updates are usually followed by an initramfs rebuild which is managed by your package manager.
* Edit `/etc/fstab` and remove `/` and `/boot` (Optional)
  * Do not forget to take notes on how the partitions should have been mounted.
  * Add a line containing `tmpfs / tmpfs size=8G 0 0` in case you or some program needed to remount `/`
* Reboot
  * And now you are on a RAM accelerated system! Congrats!

### Saving changes
As you are using your system. You probably would want to save changes after configuring it.
Before syncing the two disks you need to make sure you know where each partition should be mounted, this is something you ether have setup yourself or it has been setup by your distribution. Check `/etc/fstab` and take notes of where partitions should be mounted if you have not done this earlier.
You also need to install `rsync`
* Mount the root partition in `/mnt`
* Mount the boot partition where it should be (on void linux it is `/mnt/boot/efi`)
* execute `rsync -av / /mnt --exclude /mnt --exclude /dev --exclude /proc --exclude /run --exclude /sys --exclude /tmp`
And with that, you have saved changes to your disk. It might be a good measure to add the previous command (as well as the mount commands) to a script or an alias so that it can be executed easily when needed (the same as `lbu commit` that alpine has).

## General recommendations
It might be better to use musl for this since it takes way less space, and thus it will have less boot time and less RAM usage.
