# Redundant Array of Inexpensive Disks (RAID)

Raid contains groups or sets or Arrays. A combine of drivers make a group of disks to form a RAID Array or RAID set. Only one Raid level can be applied in a group of disks. According to our selected raid level, performance will differ. Saving our data by fault tolerance & high availability.

- __Software RAID__ have low performance, because of consuming resource from hosts. Raid software need to load for read data from software raid volumes. Before loading raid software, OS need to get boot to load the raid software. No need of Physical hardware in software raids. Zero cost investment.

- __Hardware RAID__ have high performance. They are dedicated RAID Controller which is Physically built using PCI express cards. It won’t use the host resource. They have NVRAM for cache to read and write. Stores cache while rebuild even if there is power-failure, it will store the cache using battery power backups. Very costly investments needed for a large scale.

RAID’s are in various Levels. Here we will see only the RAID Levels which is used mostly in real environment:
- __RAID 0__ = Striping
- __RAID 1__ = Mirroring
- __RAID 5__ = Single Disk Distributed Parity
- __RAID 6__ = Double Disk Distributed Parity
- __RAID 10__ = Combine of Mirror & Stripe. (Nested RAID)

RAID are managed using `mdadm` package in most of the Linux distributions. Let us get a Brief look into each RAID Levels.

### RAID 5 -- Distributed Parity
RAID 5 is mostly used in enterprise levels. RAID 5 work by distributed parity method. Parity info will be used to rebuild the data. It rebuilds from the information left on the remaining good drives. This will protect our data from drive failure. Assume we have 4 drives, if one drive fails and while we replace the failed drive we can rebuild the replaced drive from parity informations. Parity information’s are Stored in all 4 drives, if we have 4 numbers of 1TB hard-drive. The parity information will be stored in 256GB in each drivers and other 768GB in each drives will be defined for Users. RAID 5 can be survive from a single drive failure, if more than 1 drive fails, it will cause loss of data.

### RAID 6 -- Two Parity Distributed Disk
RAID 6 is same as RAID 5 with two parity distributed systems. Mostly used in a large number of arrays. We need minimum 4 drives, even if 2 drive fail, we can rebuild the data while replacing the drives. Slower than RAID 5 because it writes data to all 4 drivers at same time. Will be average in speed while we using a Hardware RAID Controller. If we have 6 numbers of 1TB hard-drives, 4 drives will be used for data and 2 drives will be used for parity.

Reference: [Introduction to RAID](https://www.tecmint.com/understanding-raid-setup-in-linux/)
___
# Windows 10 VM
1. Setup the VirtualBox to use 2 adapters:
    - The first adapter is set to NAT (for internet connection).
    - The second adapter is set to host-only.
2. Start the virtual machine and assign a static IP for the second adapter in CentOS (for instance 192.168.56.101). The host Windows will have 192.168.56.103 as IP for the internal network (VirtualBox Host-Only Network is the name in network connections in Windows). This will give access to the server on CentOS, from windows, by going to 192.168.56.101. Also, CentOS will have internet access, since the first adapter (set to NAT) will take care of that.
3. Now, to make the connection available both ways (accessing the Windows host from the CentOS guest) there's still one more step to be performed. Windows will automatically add the VirtualBox Host-Only Network to the list of public networks and that cannot be changed. This entails that the firewall will prevent proper access.
4. To overcome this and not make any security breaches in your setup:
    - go to the Windows Firewall section, in control panel;
    - click on advanced settings. In the page that pops up;
    - click on inbound rules (left column), then on new rule (right column). Chose custom rule, set the rule to allow all programs, and any protocol. For the scope, add in the first box (local IP addresses) 192.168.56.103, and in the second box (remote IP) 192.168.56.101. Click next, select allow the connection, next, check all profiles, next, give it a name and save.

This is the output of `ipconfig`:
```sh
Ethernet adapter VirtualBox NAT Adapter:
Connection-specific DNS Suffix: ##########
Link-local IPv6 Address: ####:####:####:####:####%##
Ipv4 Address: ##.##.#.##
Subnet Mask: 255.255.255.0
Default Gateway: ##.#.#.#

Ethernet adapter VirtualBox Host-Only Network:
IPv4 Address: 192.168.56.103
Subnet Mask: 255.255.255.0
Default Gateway: 192.168.56.100
```

Reference: [VirtualBox Windows Guest](https://serverfault.com/questions/225155/virtualbox-how-to-set-up-networking-so-both-host-and-guest-can-access-internet)
___

# Network File System (NFS)

All versions of NFS can use Transmission Control Protocol (TCP) running over an IP network, with NFSv4 requiring it. NFSv2 and NFSv3 can use the User Datagram Protocol (UDP) running over an IP network to provide a stateless network connection between the client and server. All NFS versions rely on Remote Procedure Calls (RPC) between clients and servers. 

### Installing
Setup a CentOS 7 machine hostname by editing the `/etc/hostname` file (this will survive reboots). A system reboot is necessary. Alternatively, you can use `hostnamectl`.

Install `mdadm` package:
```sh
sudo yum install mdadm
```
Before proceeding check the disks:
```sh
sudo lsblk
sudo fdisk -l | grep sd
```

Before creating a RAID drives, always examine (`E`) our disk drives whether there is any RAID is already created on the disks:
```sh
sudo mdadm -E /dev/sd[a-i]
sudo mdadm --examine /dev/sda /dev/sdc /dev/sdd
```

Next, create partitions for each disk. Then perform the RAID:
```sh
sudo mdadm --create /dev/md0 --level=6 --raid-devices=8 /dev/sda1 /dev/sdc1 /dev/sdd1 /dev/sde1 /dev/sdf1 /dev/sdg1 /dev/sdh1 /dev/sdi1
cat /proc/mdstat
sudo mdadm --detail /dev/md0
```

RAID doesn't have a config file by default. Use the below command and then verify the status of device `/dev/md0`:
```sh
sudo sh -c "mdadm --detail --scan --verbose >> mdadm.conf"
sudo mdadm --detail /dev/md0
```

### Incorporate the RAID6 block device into a new or existing LVM
Create the PV, then use PV to expand VG, and finally extend a LV: 
```sh
sudo pvcreate /dev/md0/
sudo pvs
sudo vgextend centos /dev/md0 
sudo pvs
sudo lvcreate -n raid6lv -l +100%FREE centos
sudo pvs
sudo lvdisplay
```
Create the shared directory `/new_home` for NFS: 
```sh
sudo mkdir /new_home
sudo mkfs -t xfs /dev/centos/raid6lv
sudo mount /dev/centos/raid6lv /new_home/
```
You can create a file in `/new_home` just to make sure it is working. Create an entry in `/etc/fstab`:
```sh
/dev/centos/raid6lv    /new_home      xfs    defaults        0 0
```
Reboot. Check with:
```sh
sudo mount -av
```
___
Reference: [Setup RAID Level 6](https://www.tecmint.com/create-raid-6-in-linux/)

# Installing NFS
On the server, install the packages, enable and start the service:
```sh
sudo yum install nfs-utils
systemctl enable nfs-server.service
systemctl start nfs-server.service
```
Install on the client as well:
```sh
sudo yum install nfs-utils
```
Next, open the SSH and NFS ports to ensure that you will be able to connect to the server by SSH for admin purposes and by NFS from NFS client (did this on the server and client machine).
```sh
sudo firewall-cmd --permanent --zone=public --add-service=ssh
sudo firewall-cmd --permanent --zone=public --add-service=nfs
sudo firewall-cmd --reload
```

### Exporting directories on the server
Modify `/etc/exports` as to "export" NFS shares (I'm giving __rw__ access to user daniela (uid=1000 and gid=1000):
```sh
/new_home 192.168.56.102(rw,anonuid=1000,anongid=1000)
```
By default, `exportfs` chooses a uid and gid of 65534 for squashed access. These values can also be overridden by the `anonuid` and `anongid` options. Finally, you can map all user requests to the anonymous uid by specifying the `all_squash` option.

Here's the complete list of mapping options:
- __root_squash__ maps requests from uid/gid 0 to the anonymous uid/gid. Note that this does not apply to any other uids or gids that might be equally sensitive, such as user bin or group staff. 
- __no_root_squash__ turns off root squashing. This option is mainly useful for diskless clients. 
- __all_squash__ maps all uids and gids to the anonymous user. The opposite option is no_all_squash, which is the default setting. 
- __anonuid and anongid__ explicitly set the uid and gid of the anonymous account. This option is primarily useful for PC/NFS clients, where you might want all requests appear to be from one user.

Load the `/etc/exports` new changes (don't know if restarting the may not be necessary):
```sh
sudo exportfs -a
sudo systemctl restart nfs
```

> Important options for `exportfs` are:
> - __-a__ export/unexport all directories;
> - __-u__ unexport one or more directories;
> - __-r__ reexport all directories, synchronizing `/var/lib/nfs/etab` with `/etc/exports` and files under `/etc/exports.d`;
> - __-v__ verbose.

### Mounting the NFS shares on the client
```sh
sudo mkdir /new_home
sudo mount 192.168.56.101:/new_home /new_home/new_home 
df -h
```
An alternate way to mount an NFS share from another machine is to add a line to the `/etc/fstab` file. The line must state the hostname of the NFS server, the directory on the server being exported, and the directory on the local machine where the NFS share is to be mounted. You must be root to modify the `/etc/fstab` file:
```sh
192.168.56.101:/new_home   /new_home  nfs     defaults    0 0    
```
After editing `/etc/fstab`, regenerate mount units so that your system registers the new configuration:
```sh
sudo systemctl daemon-reload
```

For more on NFS options, refer to [Common NFS Mount Options](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/storage_administration_guide/s1-nfs-client-config-options).

### Autofs
The `automount` utility can mount and unmount NFS file systems automatically (on-demand mounting), therefore saving system resources. First, install `autofs`:
```sh
rpm -q autofs
sudo yum install autofs
sudo systemctl start rpcbind && sudo systemctl enable rpcbind && sudo systemctl status rpcbind  
```
File `/etc/auto.master` looks like this (`\-` in the first line did the trick):
```sh
/-	/etc/auto.misc
# /net	-hosts
+dir:/etc/auto.master.d
+auto.master
```
And `/etc/auto.misc` looks like this:
```sh
/new_home -fstype=nfs,rw,soft,intr 192.168.56.101:/new_home
```
Restart autofs:
```sh
sudo systemctl restart autofs.service
sudo systemctl enable autofs.service
```
Refer to [Automount NFS share in Linux using autofs](https://www.linuxtechi.com/automount-nfs-share-in-linux-using-autofs/)

### Samba
Samba is the standard Windows interoperability suite of programs for Linux and Unix. Install Samba on the server:
```sh
sudo yum install samba samba-client samba-common
```
Create users and group:
```sh
sudo groupadd training
sudo useradd user1
sudo useradd user1
usermod -aG collaboration user1
usermod -aG collaboration user2
chmod 0770 /new_home
sudo chgrp -R training /new_home
```
Configuring SELinux and firewalld:
```sh
sudo setsebool -P samba_export_all_ro=1 samba_export_all_rw=1
sudo getsebool -a | grep samba_export
sudo yum install policycoreutils-python
sudo semanage fcontext -at samba_share_t "/new_home(/.*)?"
restorecon /new_home/
firewall-cmd --permanent --add-service=samba
firewall-cmd --reload
```
Configure samba share in `/etc/samba/smb.conf`:
```sh
[training]
	comment=Directories for group training
	browsable=yes
	path=/new_home
	public=no
	valid users=@training
	write list=@training
	writable=yes
	create mask=0770
	Force create mode=0770
	force group=training
```
Test with `testparm` utility. Next, add `user1` and `user2` as Samba users:
```sh
sudo smbpasswd -a user1
sudo smbpasswd -a user2
```
Restart:
```sh
sudo systemctl start smb
sudo systemctl enable smb
sudo smbclient -L localhost -U user1
sudo smbclient -L localhost -U user2
```
Test on CentOS client:
```sh
yum install samba samba-client samba-common cifs-utils
smbclient -L 192.168.56.101 -U user1
smbclient -L 192.168.56.101 -U user2
```
And then mount (not permanently, this is just a test):
```sh
sudo mkdir -p /mnt/samba
mount //192.168.56.101/training /mnt/samba -o username=user1
```
> IMPORTANT: what matters in Samba is the `Sharename`, not the actual directory name in the server...

### Mounting Samba on Windows 10
To mount the Samba share in Windows 10, go to `This PC` and `Map network drive`. Next, assign a letter for the drive to be mapped and check __Connect using different credentials__.
