# Set up a RHEL server

### Overview
Service startup and maintenance tool: `systemd`. Hit `cat /etc/redhat-release` to get info on RHEL release. The RHEL file system is `ext`. The default database is MariaDB. For booting from the command line, type `systemctl set-default multi-user.target` and then reboot. 

### Access subscription manager from the command line

Type `subscription-manager-gui`. Hit `subscription-manager register --username <username> --password <password>` to register. To view existing subscriptions, hit `subscription-manaher list`.

### Command line basics
Basic commands are `hostname`, `whoami` and `id -un`. Use the pipe `| less` a lot -- it's really useful! Use `head` and `tail` to display the beginning or the ending of a file.

> Grep stands for general regular expression parser

### Permissions 

Read (`r`), write (`r`) and execute (`x`) and the order these are shown is: _user_, _group_ and _everyone else_. Use the `chown` command to change ownership and `chmod` to change permissions. For example, to keep a file exclusive to its owner, type `chmod 600 <file>` to give read and write permissions only for the user.

You can find more about the `chown` command here: [Chown Command in Linux](https://linuxize.com/post/linux-chown-command/).

### Adding users that can execute `sudo`
Type `sudo /usr/sbin/visudo` to add user to sudoers configuration file. For more details, refer to [Add User to Sudoers](https://phoenixnap.com/kb/how-to-create-add-sudo-user-centos). A list of all sudoers can be found in `/etc/passwd`.

### Users and groups
Hit `useradd -m <username>` to add a new user (the `-m` oprtion creates the home directory it doesn't already exist). Hit `passwd <username>` to add a password to that user. Similarly, user `usermod` and `userdel` to modify and delete users.

> You can set the user password to expire at first login (i.e., get the user to change its password) by typing `chage -d 0 <user>`
> Use the option `-U` in `usermod` to **unlock**, i.e., remove the password lock.

The command `groupadd` adds a new group. Group info can be found in `/etc/group`. For adding a person to a group, use:
```sh
usermod -aG <group> <user>
```
the `-a` stands for **append** and `G` stands for **group**.

To assign a directory to a group of user and allow users permissions, type:
```sh
chown <user>:<group> <directory>
chmod 2775 <directory>
```

### Filesystem
Type `findmnt` -- it's readable information.

### File Management
Hit `setfacl -m u:<user>:rwx <file>` to give read, write and execute access to the user. Use `setfacl -m g:<group>:rwx <file>` to give rwx access to the group. The `-m` option is to modify the file system.

### Repos
To see currently installed repos, see the contents of `/etc/yum.repos.d/` or `yum repolist` or even `yum list extras`. You'll see the epl repo installed, for instance. To download a new repo, you can use the `wget` command, which stands for *web get*. After the download, use `rpm` command with the option `-ivh` to install:
```sh
wget <link>
rpm -ivh epel-release-7-8.noarch.rpm
```

# Template VM

In Virtual Box, set the first adapter to NAT and second adapter to Host-only adapter. 

### Configuration of network adapter

In VirtualBox, select from **File menu > Host Network Manager**. Used the following configurations:

1. Ipv4 address 192.168.56.100
2. Ipv4 network mask 255.255.255.0
3. You want the guest machines to have static IP addresses so leave DHCP **disabled**

### Set up the adapter in network settings of VM

Then, in the network settings for the VM, set up two adapters:
1. Host-only, vboxnet0
2. NAT

### Configure the VM

The command `ls /sys/class/net` provide the name of adapters on the VM (in my case, *enp0s3*, *enp0s8* and *lo*). You'll need to edit the configuration file of your host-only network adapter (enp0s3 in my case):
```sh
sudo vim /etc/sysconfig/network-scripts/ifcfg-enp0s3
```

It should look like the following:
BOOTPROTO=static
ONBOOT=yes
IPADDR=192.168.56.101
NETMASK=255.255.255.0

> The default gateway is determined by the network scripts which parse the `/etc/sysconfig/network`.

Restart the service by typing `sudo systemctl restart network.service`. Try to `ping` this IP (192.168.56.101) -- it should be OK. Now you can `ssh` the host:
```sh
ssh root@192.168.56.101
```

### Password-less SSH access
Store the public key on the remote system in a `.ssh/authorized_keys directory`. First, check that you are using OpenSSH with `ssh -V`. Use command-line SSH to generate a key pair using the RSA algorithm:
```sh
ssh-keygen -t rsa
```
you be prompted to provide a file location (default is /<home>/.ssh) and a password. 

Copy the public key to your user account using either SFTP or SCP:
```sh
scp ~/.ssh/id_rsa.pub daniela@192.168.56.101:
```

On the remote machine, add the public key to the **authorized_keys** files in `.ssh` folder (create the file if it doesn't exist):
```sh
mkdir -p /home/daniela/.ssh
touch /home/daniela/.ssh/authorized_keys
cat /home/daniela/id_rsa.pub >> /home/daniela/.ssh/authorized_keys
mv /home/daniela/id_rsa.pub /home/daniela/.ssh/
```

Then I was able to SSH to my account in the host VM:
```sh
ssh daniela@192.168.56.101
```
If the private key is in a different directory, use:
```sh
ssh -i <private_key_path> daniela@192.168.56.101
```

Reference: [Set up SSH public-key authentication to connect to a remote system](https://kb.iu.edu/d/aews)
