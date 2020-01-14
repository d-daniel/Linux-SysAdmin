# Account Security

### Allow SSH Access to a user or group
OpenSSH default configuration file has two directives for both allowing and denying SSH access to a particular user(s) or a group:
```sh
sudo vi /etc/ssh/sshd_config
```
Add or edit the following lines:
```sh
AllowUsers <user_1> <user_2> ... <user_n> 
AllowGroups <group_1> <group_2> ... <group_n>
```
To disable or deny SSH access to any user or group, add/edit the following lines:
```sh
DenyUsers <user_1> <user_2> ... <user_n> 
DenyGroups <group_1> <group_2> ... <group_n>
```
Prevent root login with this line:
```sh
PermitRootLogin no
```
Restart the service:
```sh
sudo systemctl restart sshd
```

### Add sudo ability for user
By default, on CentOS, members of the wheel group have sudo privileges:
```sh
usermod -aG wheel <user>
```
> The command `su` stands for “substitute user”.

### Editing the `/etc/sudoers` file
Improper syntax in the `/etc/sudoers` file can leave you with a system where it is impossible to obtain elevated privileges; so it is important to use the `visudo` command to edit the file. To disallow a user to run a command with sudo (`su` in this case):
```sh
<user>  ALL = NOEXEC: /bin/su
```
Another possibility:
```sh
<user>  ALL = !/bin/su
```
Disallowing a sudo user to perform commands makes no sense (there are infinite ways to circumvent this). Instead of adding the target user in the wheel group and _restrict_ his commands, just edit the file to _allow_ commands for a regular user. 
> __IMPORTANT:__ Later rules will override earlier rules when there is a conflict between the two.

Reference: [How To Edit the Sudoers File on Ubuntu and CentOS](https://www.digitalocean.com/community/tutorials/how-to-edit-the-sudoers-file-on-ubuntu-and-centos)

### Using PAM (Pluggable Authentication Modules) to limit root access to services 
To limit root access to a system service, edit the file for the target service in the `/etc/pam.d/` directory and make sure the pam_listfile.so module is required for authentication. Add the followind line __at the top__ of `/etc/pam.d/login`:
```sh
auth	   required     pam_listfile.so  item=user  sense=deny  file=/etc/login/deny  onerr=succeed
```
Understanding the config directives:
- __auth required pam_listfile.so:__ name of module required while authenticating users
- __item=user:__ check the username
- __sense=deny:__ deny user if existing in specified file
- __file=/etc/login/deny:__ name of file which contains the list of user (one user per line)
- __onerr=succeed:__ if an error is encountered PAM will return status PAM_SUCCESS

Reference: [Controlling Root Access](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/security_guide/sec-controlling_root_access)

# Firewalld
In order from least trusted to most trusted, the predefined zones within __firewalld__ are: __drop__, __block__, __public__, __external__, __internal__, __dmz__, __work__, __home__ and __trusted__. Rules can be designated as either _permanent_ or _immediate_. Most `firewall-cmd` operations can take the `--permanent` flag to indicate that the non-ephemeral firewall should be targeted. 
Verify that the service is running and reachable by typing:
```sh
sudo firewall-cmd --state
```
Verify the active zones:
```sh
firewall-cmd --get-active-zones
```
Set default zone:
```sh
sudo firewall-cmd --set-default-zone=home
```
List available services (or got to `/usr/lib/firewalld/services` for more details):
```sh
firewall-cmd --get-services
```
Enable a service for a zone:
```sh
sudo firewall-cmd --zone=public --add-service=http
```
List the services of a zone:
```sh
sudo firewall-cmd --zone=public --list-services
```
To opening a port for a zone (protocols can be either TCP or UDP):
```sh
sudo firewall-cmd --zone=public --add-port=5000/tcp
```
To add a range of ports:
```sh
sudo firewall-cmd --zone=public --permanent --add-port=4990-4999/udp
```
Services are simply collections of ports with an associated name and description. Using services is easier to administer than ports. To create a service, copy an existing script from `/usr/lib/firewalld/services` to `/etc/firewalld/services`.
Reload the firewall:
```sh
sudo firewall-cmd --reload
```
> The predefined zones will probably be more than enough, but it is possible to define new zones just in case.

Reference: [How To Set Up a Firewall Using FirewallD on CentOS 7](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-using-firewalld-on-centos-7)

### Restrict SSH access to IP
Using _firewalld_:
```sh
sudo firewall-cmd --add-rich-rule 'rule family="ipv4" service name="ssh" source address="192.168.56.100" accept' --permanent
```
Using TCP wrappers: edit the files `/etc/hosts.deny` and `/etc/hosts.allow`.
Using SSH daemon configuration: add desired authentication methods after a `Match Address` in `sshd_config`. 
See: [Limit SSH access to specific clients by IP address](https://unix.stackexchange.com/questions/406245/limit-ssh-access-to-specific-clients-by-ip-address)
Block connections on ports 22 and 80 to your machine from a machine with a local IP address 192.168.0.29:
```ssh
sudo firewall-cmd --zone=internal --add-rich-rule 'rule family="ipv4" source address="192.168.0.19" port port=22 protocol=tcp reject'
sudo firewall-cmd --zone=internal --add-rich-rule 'rule family="ipv4" source address="192.168.0.19" port port=80 protocol=udp reject'
```

### Setup a project using protected data
User _daniela_ is the owner of `/home/project/`. Users _rocket_ and _warrior_ have access to this folder. User _dragon_ is in the same group as _daniela_, but cannot access `home/project/`. The ACLs of `/home/project/` are as follows:
```sh
$ sudo getfacl /home/project/
# file: project/
# owner: daniela
# group: daniela
user::rwx
user:dragon:---
group::rwx
mask::rwx
other::r-x
```