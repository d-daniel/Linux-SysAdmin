# OpenHPC
The OpenHPC _master_ node is also the NFS server (IP 192.168.56.101) and the 4 compute nodes will have IPs 192.168.56.140-143. Each compute node shall have access to the `\new_home` directory from the NFS server with __no_root_squash__ option enabled.
___

### Prerequisites
Disable SELinux on the _master_. Open the `/etc/selinux/config` file and set the `SELINUX` mode to `disabled`.

> Even the use of permissive mode can be problematic and we therefore recommend disabling SELinux on the master host. Reboot. Check that SELinus is disabled with the `sestatus` command. 

Disable the firewall service as well:
```sh
systemctl disable firewalld
systemctl stop firewalld
```
### Install OpenHPC Components
Install OpenHPC release package directly from the OpenHPC build server:
```sh
sudo yum install http://build.openhpc.community/OpenHPC:/1.3/CentOS_7/x86_64/ohpc-release-1.3-1.el7.x86_64.rpm
```
Install base meta-packages:
```sh
sudo yum install ohpc-base
sudo yum install ohpc-warewulf
```
Set network booting as the primary option in the boot order by default on all nodes. 

### Setting Up “NTP (Network Time Protocol) Server”
```sh
sudo yum install ntp
```
Edit file `/etc/ntp.conf`. Mine is something like this:
```sh
# Hosts on local network are less restricted.
restrict 192.168.56.100 mask 255.255.255.0 nomodify notrap nopeer

# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (http://www.pool.ntp.org/join.html).
server 0.us.pool.ntp.org iburst
server 1.us.pool.ntp.org iburst
server 2.us.pool.ntp.org iburst
server 3.us.pool.ntp.org iburst

includefile /etc/ntp/crypto/pw

# Key file containing the keys and key identifiers used when operating
# with symmetric key cryptography. 
keys /etc/ntp/keys

disable monitor

logfile /var/log/ntp.log
```

Enable the service:
```sh
sudo systemctl start ntpd
sudo systemctl status ntpd
sudo systemctl enable ntpd
```

### Setting Up “NTP (Network Time Protocol) Client”
Install the service and edit file `/etc/ntp.conf`:
```sh
server centOSserver.training.edu 
```
Start and enable the service. Test with `ntpq -p`.

Reference: [Setting Up “NTP (Network Time Protocol) Server” in RHEL/CentOS 7](https://www.tecmint.com/install-ntp-server-in-centos/)

### Adding Slurm workload manager server components to the chosen master host
Install slurm server meta-package:
```sh
sudo yum install ohpc-slurm-server
```
Edit file `/etc/slurm/slurm.conf` to configure your cluster. Mine looks like this:
```sh
ControlMachine=centosserver.training.edu
NodeName=compnode[0-3] Sockets=1 CoresPerSocket=4 ThreadsPerCore=1 State=UNKNOWN
PartitionName=normal Nodes=compnode[0-3] Default=YES MaxTime=24:00:00 State=UP
```

### Complete basic Warewulf setup for master system management server (SMS) node
Configure Warewulf provisioning to use desired internal interface:
```sh
perl -pi -e "s/device = eth1/device = enp0s3/" /etc/warewulf/provision.conf
```
Enable tftp service for compute node image distribution:
```sh
perl -pi -e "s/^\s+disable\s+= yes/ disable = no/" /etc/xinetd.d/tftp
```
Enable internal interface for provisioning (if not enabled yet):
```sh
ifconfig enp0s3 192.168.56.101 netmask 255.255.255.0 up
```
Restart and/or enable services:
```sh
sudo systemctl restart xinetd
sudo systemctl enable mariadb
sudo systemctl restart mariadb
sudo systemctl enable httpd
sudo systemctl restart httpd
sudo systemctl enable dhcpd
```

### Build initial Basic Operating System image
With the provisioning services enabled, the next step is to define and customize a system image that can subsequently be used to provision one or more compute nodes.

Define `chroot` location:
```sh
export CHROOT=/opt/ohpc/admin/images/centos7
```
Build initial `chroot` image:
```sh
sudo wwmkchroot centos-7 $CHROOT
```
The `wwmkchroot` process used in the previous step is designed to provide a minimal CentOS 7.7 configuration. Next, we add additional components to include resource management client services, NTP support, and other additional packages to support the default OpenHPC environment.

Install compute node base meta-package:
```sh
sudo yum --installroot=$CHROOT install ohpc-base-compute
```
Update `chroot` environment's DNS:
```sh
sudo cp -p /etc/resolv.conf $CHROOT/etc/resolv.conf
```
Include additional components (Slurm, NTP, kernel drivers and modules user environment) to the compute instance using `yum` to install into the `chroot`:
```sh
sudo yum --installroot=$CHROOT install ohpc-slurm-client
sudo yum --installroot=$CHROOT install ntp
sudo yum --installroot=$CHROOT install kernel
sudo yum --installroot=$CHROOT install lmod-ohpc
```
Include LDAP client on both SMS and images:
```sh
sudo yum --installroot=$CHROOT openldap-clients 
sudo yum --installroot=$CHROOT nss-pam-ldapd
```
Ensure that you can authenticate against the SMS via `nslcd` service. Test if can query the database, e.g.:
```sh
sudo ldapsearch -H ldaps://ldapserver.training.edu -D cn=overlord,ou=system,dc=training,dc=edu -W -b uid=s191001,ou=users,dc=training,dc=edu
```
Copy all `nslcd` and LDAP related files to `$CHROOT`:
```sh
sudo cp -p /etc/nslcd.conf $CHROOT/etc/nslcd.conf
sudo cp -p /etc/openldap/ldap.conf $CHROOT/etc/openldap/ldap.conf
```

Note this 2 lines at the end of `/etc/passwd`:
```sh
nscd:x:28:28:NSCD Daemon:/:/sbin/nologin
nslcd:x:65:55:LDAP Client User:/:/sbin/nologin
```

### Customize system environment

Create a user (wwuser) and a password (wwuser) in `/etc/warewulf/database.conf`. Create that user in MariaDB with the command:
```sh
CREATE USER wwuser@localhost IDENTIFIED BY 'wwuser';
```
It should be OK to initialize (had to change the script to look for __MariaDB-server__):
```sh
sudo wwinit -v database
```

Execute `authconfig` on `$CHROOT`:
```sh
sudo chroot $CHROOT authconfig --enableldap --enableldapauth --ldapserver=ldaps://ldapserver.training.edu --ldapbasedn="dc=training,dc=edu" \
--enablemkhomedir --update
```

```sh
# Initialize warewulf ssh_keys
sudo wwinit database
sudo wwinit ssh_keys
# Add NFS client mounts to base image
echo "192.168.56.101:/new_home /home nfs defaults 0 0" >> $CHROOT/etc/fstab
# Enable NTP time service on computes and identify master host as local NTP server
sudo chroot $CHROOT systemctl enable ntpd
echo "server 192.168.56.101" >> $CHROOT/etc/ntp.conf
sudo chroot $CHROOT systemctl enable nslcd
```
To import local file-based credentials, issue the following:
```sh
sudo wwsh file import /etc/passwd
sudo wwsh file import /etc/group
sudo wwsh file import /etc/shadow
```

Grant `wwuser` access to database. Log as root into MariaDB and issue the following commands:
```sh
GRANT INSERT ON *.* TO 'wwuser'@'localhost';
GRANT DELETE ON *.* TO 'wwuser'@'localhost';
GRANT UPDATE ON *.* TO 'wwuser'@'localhost';
```

To import the global Slurm configuration file and the cryptographic key that is required by the _munge_ authentication library to be available on every host in the resource management pool, issue the following:
```sh
sudo wwsh file import /etc/slurm/slurm.conf
sudo wwsh file import /etc/munge/munge.key
```

### Assemble bootstrap image
```sh
# (Optional) Include drivers from kernel updates; needed if enabling additional kernel modules on computes
export WW_CONF=/etc/warewulf/bootstrap.conf
echo "drivers += updates/kernel/" >> $WW_CONF
# (Optional) Include overlayfs drivers; needed by Singularity
echo "drivers += overlay" >> $WW_CONF
# Build bootstrap image
sudo wwbootstrap `uname -r`
```

### Assemble Virtual Node File System (VNFS) image
```sh
sudo wwvnfs --chroot $CHROOT
```

### Register nodes for provisioning
Define the desired network settings for the compute nodes with the underlying provisioning system and restart the `dhcp` service. Use variable names for the desired compute hostnames, node IPs, and MAC addresses which should be modified to accommodate local settings and hardware. The final step in this process associates the VNFS image assembled in previous steps with the newly defined compute nodes, utilizing the user credential files and _munge_ key.
```sh
# Set provisioning interface as the default networking device
echo "GATEWAYDEV=enp0s3" > /tmp/network.$$
sudo wwsh -y file import /tmp/network.$$ --name network
sudo wwsh -y file set network --path /etc/sysconfig/network --mode=0644 --uid=0
# Add nodes to Warewulf data store
sudo wwsh -y node new compnode0 --ipaddr=192.168.56.140 --hwaddr=08:00:27:B5:83:A1 -D enp0s3
sudo wwsh -y node new compnode1 --ipaddr=192.168.56.141 --hwaddr=08:00:27:1D:22:AD -D enp0s3
sudo wwsh -y node new compnode2 --ipaddr=192.168.56.142 --hwaddr=08:00:27:94:01:27 -D enp0s3
sudo wwsh -y node new compnode3 --ipaddr=192.168.56.143 --hwaddr=08:00:27:42:2C:71 -D enp0s3
# Required for "enp0s3" interface
export kargs="${kargs} net.ifnames=1,biosdevname=1"
sudo wwsh provision set --postnetdown=1 "comp*"
# Define provisioning image for hosts
sudo wwsh -y provision set "comp*" --vnfs=centos7 --bootstrap=`uname -r` --files=dynamic_hosts,passwd,group,shadow,slurm.conf,munge.key,network
```

Restart `dhcpd` and update PXE:
```sh
sudo systemctl restart dhcpd
sudo wwsh pxe update
```

### Boot compute nodes
> __Install the extension pack for VirtualBox.__ 

At this point, the master server should be able to boot the newly defined compute nodes. Assuming that the compute node BIOS settings are configured to boot over PXE...
[![How PXE works](https://gerardnico.com/_media/virtualbox/pxe_boot_install.png?ezimgfmt=rs:399x620/rscb1/ngcb1/notWebP)](https://gerardnico.com/virtualbox/pxe)
Check that `nslcd` has started smoothly and log in using LDAP (file `/etc/passwd` has to have users `nscd` and `nslcd`).

### OpenHPC Development Components
Add OpenHPC-provided packages to support a flexible HPC development environment including development tools, C/C++ compilers, MPI stacks, etc. OpenHPC provides recent versions of the GNU autotools collection, the Valgrind memory debugger, EasyBuild, and Spack:
```sh
sudo yum install ohpc-autotools
sudo yum install EasyBuild-ohpc
sudo yum install hwloc-ohpc
sudo yum install spack-ohpc
sudo yum install valgrind-ohpc
```
### Compilers and MPI Stacks
```sh
sudo yum install gnu8-compilers-ohpc
sudo yum install llvm5-compilers-ohpc
sudo yum install openmpi3-gnu8-ohpc mpich-gnu8-ohpc
```
### Perf Tools
```sh
sudo yum install ohpc-gnu8-perf-tools
```

### Default development environment
```sh
sudo yum install lmod-defaults-gnu8-openmpi3-ohpc
```

> You'll have to install those things to the `chroot` environment, obviously.

### Startup
```sh
# Start munge and slurm controller on master host
sudo systemctl enable munge
sudo systemctl enable slurmctld
sudo systemctl start munge
sudo systemctl start slurmctld
# Start slurm clients on compute hosts
sudo pdsh -w compnode[0-3] systemctl start slurmd
# You may have to manually start the nodes
sudo scontrol update nodename=compnode[0-3] state=resume
# Check that all nodes are idle
sinfo
```

### Passwordless SSH access
Generate a key pair using the RSA for each user that's going to submit jobs to the cluster. Store the public key on the remote system in a `.ssh/authorized_keys` directory (which in my case is exported through NFS). As one such user, you should be able to `ssh` each compute node from the SMS. 

### Submit a test job
OpenHPC includes a simple “hello-world” MPI application in the `/opt/ohpc/pub/examples/mpi` directory that can be used for this quick compilation and execution. OpenHPC also provides a companion job-launch utility named prun that is installed in concert with the pre-packaged MPI toolchains. I have a simple slurm script `myjob` to launch `hello`:
```sh
#!/bin/bash

#SBATCH --job-name=hello
#SBATCH --output=hello.txt
#SBATCH --ntasks=16
#SBATCH --time=10:00
#SBATCH --nodes=4

prun ./hello
```
Now compile and run the script:
```sh
# Compile MPI "hello world" example
[s191001@centosserver ~]$ cp /opt/ohpc/pub/examples/mpi/hello.c .
[s191001@centosserver ~]$ mpicc -O3 -o hello hello.c
# Symbolic link to the NFS shared directory
[s191001@centosserver ~]$ ln -s /new_home/s191001 shared
[s191001@centosserver ~]$ cp hello shared
[s191001@centosserver ~]$ sbatch myjob
```

Alternatively, you can use iterative mode:
```sh
[s191001@centosserver ~]$ srun -n 8 -N 2 --pty /bin/bash
bash-4.2$ prun ./hello
```












