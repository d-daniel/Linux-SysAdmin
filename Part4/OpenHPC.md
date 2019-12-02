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
ControlMachine=centOSserver.training.edu
NodeName=c[1-4] Sockets=1 CoresPerSocket=4 ThreadsPerCore=1 State=UNKNOWN
```

### Complete basic Warewulf setup for master node
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