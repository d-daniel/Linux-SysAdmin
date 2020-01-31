# Linux Stuff

### General Stuff
- `pwd` stands for Print Working Directory
- the `root` directory is denoted by a single slash (`/`)
- `~` (tilde) is a shortcut for your home directory
- `.` (dot) is a reference to your current directory
- `..` (dotdot) is a reference to the parent directory
- `mkdir` is short for "make directory"
- use `touch` to create a blank file

>In Linux, extensions are not important. However, there is a command called `file` to help find out what type of file it is. Also, if you see space in names, just use quotes or an escape character, which is a backslash (`\`). For example:
```sh
mkdir bla\ bla
cd bla\ bla/ 
```

To create multiple directories:
```sh
mkdir {dir1,dir2,dir3}
```

You can use `tee` to redirect output to a file and to the screen at the same time:
```sh
<command> | tee <file>
```
Use the option `-a` to append to an existing file:
```sh
<command> | tee -a <file>
```

Useful options for grep command:

| Option | Description |
| ------ | ------ |
| -i | ignore case |
| -v | invert search |
| -c | count match results |
| -o | only characters that match |
| -r | read files recursively |
| -E | use Extedn regular expressions |

Use `journalctl` for accessing non-persistent logs. Examples:
```sh
journalctl --since "YYYY-MM-DD HH:MM:SS" --until "YYYY-MM-DD HH:MM:SS"
journalctl --since yesterday
journalctl --since 09:00 --until "1 hour ago"
journalctl -u nginx.service
journalctl _UID=33 --since today
```

### Hidden files
If a file has its name beggining with a `.`, then that's a hidden file. Hit `ls -a` to list hidden files as well.

### Man pages
Searching within a manual page is possible. Whilst you are in the particular manual page press forward slash `/` followed by the term you would like to search for and hit `Enter`. If the term appears multiple times you may cycle through them by pressing the `n` button for next or the `p` button.

### Copying and moving files
When we use `cp` the destination can be a path to either a file or directory. If it is to a file then it will create a copy of the source but name the copy the filename specified in destination. Normally, the command `mv` will be used to move a file or directory into a new directory. We may provide a new name for the file or directory and as part of the move it will also rename it. If we specify the destination to be the same directory as the source, but with a different name, then we have effectively used `mv` to rename a file or directory.

# Creating a CentOS VM

### Getting started
On Virtual Box, go to **File > Preferences > Input** and click on Virtual Machine tab to get a list of useful shortcuts. Once you've created a new VM, check "Enable I/O APIC", "Enable PAE/NX",  "Enable VT-x/AMD-V", and "Enable Nested Paging" so that the VM will run faster. Don't forget to give some processors to the VM. Enable 3D Acceleration and give some more video memory to your VM.

>You can create other disks for your VM in the Storage tab.

In the Network tab, the default adapter is NAT, which allows access to the internet. To remotely access the VM, you'll need a Bridged Adapter.

### Clone Virtual Machine

That way, you don't need to set the optmizations again. *I had Virtual Box generate new MAC addresses for all network adapters*.
> I set "admin<xxxxxx>" as the root password. For the first user (make sure to make this user administrator), the password is the same.

**IMPORTANT:** go to system tab and change the boot-order like in the below screenshot.

### What is yum?
Yum. Yellow dog Updater, Modified (Yum) is the default package manager used in CentOS (all versions). It is used to install and update packages from CentOS (and 3rd party) repositories.

In order to update yum, configure a proxy first:
```sh
echo "export http_proxy=http://<proxy>:<port>" > /etc/profile.d/proxy.sh
echo "export https_proxy=http://<proxy>:<port>" >> /etc/profile.d/proxy.sh
echo "proxy=http://<proxy>:<port>" >> /etc/yum.conf
```

After that you'll be able to update yum:
```sh
yum update
```

To list all installed packages and updates available:
```sh
yum list installed
yum list updates
```

A new and useful command is `group`, like this:
```sh
yum group list
yum group info "Development Tools"
```

### VERY IMPORTANT
To undo a `yum` command (useful for removing dependencies), type:
```sh
yum history
yum history undo <ID>
```

### Install Guest Additions
Once you've updated yum and your user is an administrator, type:
```sh
sudo yum groupinstall "Development Tools" # "Development Libraries"
```
Install EPEL repo (required) like this:
```sh
sudo yum -y install epel-release
```
You shall be able to see installed repos using:
```sh
sudo yum repo list 
```
Just to make sure, install the prerequisites:
```sh
sudo yum install perl gcc dkms kernel-devel kernel-headers make bzip2
```
Confirm that kernel-headers installed are matching your currently running kernel:
```sh
ls -l /usr/src/kernels/$(uname -r)
```
Click on "Devices" tab on Virtual Box and "Insert Guest Additions CD image". After that you'll have to mount the media manually:
```sh
blkid
```
The relevant block device is something like `/dev/sr0`. Create a mount point and then mount:
```sh
mkdir /media/iso
mount /dev/sr0 /media/iso/
```
or
```sh
mkdir /media/iso
mount /dev/cdrom /media/iso/
```

This is a cool script to set the resolution: [xrandr.sh](https://gist.github.com/chirag64/7853413)

**Reference**: [How To Install EPEL Repo on a CentOS and RHEL 7.x](https://www.cyberciti.biz/faq/installing-rhel-epel-repo-on-centos-redhat-7-x/)
___
For details, see [VirtualBox Guest Additions on Fedora 30/29, CentOS/RHEL 8/7/6/5](https://www.if-not-true-then-false.com/2010/install-virtualbox-guest-additions-on-fedora-centos-red-hat-rhel/).

If running a minimal installation of Linux, you have to hit `--nox11`. Nonetheless, the file `vboxadd-setup.log` will show the following error:
```console
Could not find the X.Org or XFree86 Window System, skipping. 
```

If you hit `systemctl -a | grep box`, you'll notice there is no `vboxadd-x11.service`. So, to install the guest additions anyway, use the command:
```sh
sudo ./VBoxLinuxAdditions.run --nox11 
```

I used the command `lsmod | grep -i vbox` to check guest additions were installed. To unmount the iso hit `sudo umount /devsr0` and eject the media using `eject /dev/sr0`. 

### Introduction to systemd

In Linux, a system service is called a `daemon`. Show unit service files by typing:
```sh
systemctl list-unit-files -at service
```
To show enabled running services: 
```sh
systemctl list-units -at service
```

You can manage services using `sudo systemctl start/stop/status/enable/disable <service>`. Use `timedatectl` for time service. You can manage one-time jobs with `at` and reoccurring user jobs with `cron`.

> Use `man crontab` and `man 5 crontab` for info on `cron` and its time format.

### Using Network Manager from CLI

1) The first step, you must set 'NAT' connection in virtualbox. 
2) Then, you're going to use a command line tool for controlling NetworkManager. Type the following:
```sh
nmcli d
nmtui
```
3) Restart network with command : service network restart 
4) Restart centos7 with command : sudo reboot now

---

Install vscode: [How to Install Visual Studio Code on CentOS 7](https://linuxize.com/post/how-to-install-visual-studio-code-on-centos-7/)
