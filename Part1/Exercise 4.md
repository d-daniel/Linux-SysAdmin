# Exercise 4

### Logical Volume Manager (LVM)

LVM allows for very flexible disk space management. It provides features like the ability to add disk space to a logical volume (LV) and its filesystem while that filesystem is mounted and active and it allows for the collection of multiple physical hard drives and partitions into a single volume group (VG) which can then be divided into logical volumes. Regular file systems, such as EXT3 or EXT4, can then be created on a logical volume.

[![fig1](https://opensource.com/sites/default/files/resize/images/life-uploads/lvm-520x222.png)](https://opensource.com/business/16/9/linux-users-guide-lvm)

### Logical Unit Number (LUN)

A LUN is a construct, usually on a storage array, with which you present a "slice" of a disk array/volume to a host, where it appears as a physically attached local disk via some connection, usually small computer system interface (SCSI), internet SCSI (iSCSI). An SCSI LUN can be addressed with a combination of the controller ID, the target ID, a disk ID, and occasionally the slice ID. The identifications (IDs) in a UNIX OS are generally joined as one word.

A typical example is the address c1t2d3s4. This refers to controller 1, target 2, disk 3 and slice 4. Full device addresses are as follows:
- c-part: controller ID of host bus adapter
- t-part: target ID classifying SCSI target on the bus
- d-part: disk ID classifying LUN on the target
- s-part: slice ID classifying exact slice on the disk

### Host Bus Adapter (HBA)

A host bus adapter (HBA) provides input/output (I/O) processing and physical connectivity between a host system, or server, and a storage and/or network device. Because an HBA typically relieves the host microprocessor of both data storage and retrieval tasks, it can improve the server's performance time. An HBA and its associated disk subsystems are sometimes referred to as a disk channel (see [HBA](https://searchstorage.techtarget.com/definition/host-bus-adapter)).
___
Power off the system and create a new Virtual Hard Disk in VirtualBox. Then boot again.
Useful commands:
```sh
cat /proc/partitions
lsblk
```
See all partitions in volume `sda`:
```sh
sudo fdisk -l /dev/sda
```
> Tools for managing partitions are: `fdisk`, `gdisk` and `parted`. 

In VirtualBox, I've added a new SATA disk (sdb) with 1GB of space. I'll use `gdisk` to add a new partition and write the partition table to disk:
```sh
sudo gdisk /dev/sdb
```
Then hit `?`, `n`, `Enter` (x4), `w` and `y`. Check the new partition:
```sh
cat /proc/partitions
``` 
To create an LVM system, add a partition or drive to a volume group an divide that VG into LVs. To make the partition created a physical volume (PV):
```sh
sudo pvcreate /dev/sdb1
sudo pvdisplay
```
Now, create a volume group and include the PV:
```sh
sudo vgcreate <vgname> /dev/sdb1
sudo vgs
sudo vgdisplay
```
Create a LV in this VG (you can also delete an LV with `lvremove`):
```sh
sudo lvcreate --name <lvname> --size <size>M <vgname>
sudo lvdisplay
```
To format the LV as `xfs`:
```sh
sudo mkfs -t xfs /dev/<vgname>/<lvname>
sudo blkid
```
Create a mount point and mount it:
```sh
sudo mkdir /media/<lvname>
sudo mount /dev/<vgname>/<lvname> /media/<lvname>/
df -T
mount
```
Edit the filesystem table:
```sh
sudo vim /etc/fstab
```
Append this line to the file:
```sh
/dev/<vgname>/<lvname>       /media/<lvname>   xfs     defaults        0 0
```
Unmount:
```sh
sudo umount /media/<lvname>
sudo mount -a
```
And reboot. 
I've added a new SATA disk (sdc) with 1GB of space and used `gdisk` to create a new partition.
```sh
sudo gdisk /dev/sdc
```
```sh
cat /proc/partitions
sudo pvcreate /dev/sdc1
sudo pvs
sudo vgextend <vgname> /dev/sdc1
sudo vgs
sudo lvs
sudo lvcreate --name <newlvname> --size 500M <vgname>
sudo lvs
sudo lvdisplay
sudo mkfs -t xfs /dev/<vgname>/<newlvname>
sudo blkid
sudo mkdir /media/<newlvname>
sudo mount /dev/<vgname>/<newlvname> /media/<newlvname>/
```
Edit the filesystem table:
```sh
sudo vim /etc/fstab
```
Append this line to the file:
```sh
/dev/<vgname>/<newlvname>       /media/<newlvname>   xfs     defaults        0 0
```
Umount:
```sh
sudo umount /media/<newlvname>/
```
Now extend the new LV to use all free space (the `+` symbol makes all the difference):
```sh
sudo lvresize -l +100%FREE /dev/<vgname>/<newlvname>
df -h
sudo xfs_growfs /dev/<vgname>/<newlvname>
```

___
__NOTE:__ `/mnt` is a better place to `mount`. Folder `/media` is mostly reserved for other stuff. 