# Exercise 1

Check current kernel version and available updades: 
```sh
sudo yum list installed kernel-*
uname -r
sudo yum list available kernel
```

### Updating the kernel to the latest version
The first thing we must do before upgrading the kernel is to upgrade all packages (__not necessary__) to the latest version: 
```sh
yum -y update
yum -y install yum-plugin-fastestmirror
cat /etc/centos-release
cat /etc/os-release
```
To enable and install [ELRepo](http://elrepo.org/tiki/tiki-index.php), import the public key and install:
```sh
sudo rpm --httpproxy http://<proxy>:<port> --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
yum install https://www.elrepo.org/elrepo-release-7.0-4.el7.elrepo.noarch.rpm
yum repolist
```

>ELrepo is an RPM repository for Enterprise Linux packages. ELRepo supports Red Hat Enterprise Linux (RHEL) and its derivatives (Scientific Linux, CentOS & others). The ELRepo Project focuses on hardware related packages to enhance your experience with Enterprise Linux. This includes filesystem drivers, graphics drivers, network drivers, sound drivers, webcam and video drivers.

Type the following `yum` command to list all packages in elrepo-kernel repo:
```sh
yum list available --disablerepo='*' --enablerepo=elrepo-kernel
```
You can install **long term support kernel**
```sh
yum --disablerepo='*' --enablerepo=elrepo-kernel install kernel-lt
```
or **mainline stable kernel**
```sh
yum --disablerepo='*' --enablerepo=elrepo-kernel install kernel-ml
```
and reboot. You'll notice the new kernel:
```sh
cat /proc/version
```

### Apply security updates
If **yum-plugin-security** is not installed, install it using below command:
```sh
yum install yum-plugin-security
yum updateinfo list security all
yum updateinfo list security installed
```
To install all security packages:
```sh
yum update --security
```

Reference: [Installing Security Vulnerabilities with yum](https://www.thegeekdiary.com/installing-security-vulnerabilities-with-yum-on-centos-rhel-567-cheat-sheet/)

### Add EPEL and MySQL repos and install
Install EPEL repo (required) like this:
```sh
sudo yum -y install epel-release
```
Download MySQL with `wget`:
```sh
sudo wget https://dev.mysql.com/get/mysql80-community-release-el7-3.noarch.rpm
```
> You might need to set the proxy for `wget` (all users, I  guess) by editing the file `/etc/wgetrc`.

Check the MD5 value to authenticate the software:
```sh
sudo md5sum mysql80-community-release-el7-3.noarch.rpm
```
The system should respond with a long string of letters and numbers. To update the software repositories, use the command:
```sh
sudo rpm --httpproxy http://<proxy>:<port> -ivh mysql80-community-release-el7-3.noarch.rpm
```
Install MySQL Server:
```sh
sudo yum install mysql-server
```

Reference: [How To Install MySQL on CentOS 7](https://phoenixnap.com/kb/how-to-install-mysql-on-centos-7)

### Remove MySQL and replace it with MariaDB
Uninstall MySQL Packages:
```sh
sudo yum remove mysql mysql-server mysql-user
```
Romove MySQL directories in `/var/lib/mysql` and `/etc/mysql` if those exist.

Enable the MariaDB repository. Create a repository file named MariaDB.repo and add the following content:
```sh
# MariaDB 10.3 CentOS repository list - created 2018-05-25 19:02 UTC
# http://downloads.mariadb.org/mariadb/repositories/
[mariadb]
name = MariaDB
baseurl = http://yum.mariadb.org/10.3/centos7-amd64
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1
```
Install the MariaDB server and client packages using yum, same as other CentOS package:
```sh
sudo yum install MariaDB-server MariaDB-client
```
Once the installation is complete, enable MariaDB to start on boot and start the service:
```sh
sudo systemctl enable mariadb
sudo systemctl start mariadb
sudo systemctl status mariadb
```
Reference: [Install MariaDB on CentOS 7](https://linuxize.com/post/install-mariadb-on-centos-7/)

### Installing Singularity 3 (and Go Programming Language)

Download the Go binary (refer to [how to install Go on CentOS 7](https://linuxize.com/post/how-to-install-go-on-centos-7/)):
```sh
wget https://golang.org/dl/go1.13.1.linux-amd64.tar.gz
```
Verify the tarball:
```sh
wget https://golang.org/dl/go1.13.1.linux-amd64.tar.gz
sha256sum go1.13.1.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.13.linux-amd64.tar.gz
```
Update the path by adding the following line to `/etc/profile` and load the new PATH environment variable into the current shell session:
```sh
export PATH=$PATH:/usr/local/go/bin
source /etc/profile
```

Optionally, instead of editing `/etc/profile`, go for `.bashrc` or `.bash_profile` and don't forget to run `source ~/.bashrc`.
Test if go is working properly (I'll not go over the details here)...
___

Install dependancies:
```sh
sudo yum update -y && \
sudo yum groupinstall -y 'Development Tools' && \
sudo yum install -y \
openssl-devel \
libuuid-devel \
libseccomp-devel \
wget \
squashfs-tools
```
Return to your `go` directory. If you are installing Singularity 3 you will also need to install `dep` for dependency resolution:
```sh
go get -u github.com/golang/dep/cmd/dep
```
I will install Singularity from the GitHub repo to `/usr/local`:
```sh
go get -d github.com/sylabs/singularity
```
Go will complain that there are no Go files, but it will still download the Singularity source code to the appropriate directory within the `src`.
```sh
export VERSION=v3.0.3 && \
cd /home/daniela/go/src/github.com/sylabs/singularity && \
git fetch && \
git checkout $VERSION
```
Notice that Singularity uses a custom build system called `makeit`. `mconfig` is called to generate a `Makefile` and then `make` is used to compile and install.
```sh
./mconfig && \
make -C ./builddir && \
sudo make -C ./builddir install
```
Hit `. /usr/local/etc/bash_completion.d/singularity` to use bash completion file.
Reference: [https://sylabs.io/guides/3.0/user-guide/installation.html](https://sylabs.io/guides/3.0/user-guide/installation.html)
