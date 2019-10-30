# Exercise 2

### User and group accounts
```sh
sudo groupadd surgery
sudo useradd -g surgery henry
sudo useradd -g surgery maria
id henry
id maria
sudo groupadd psychiatry
sudo useradd -g psychiatry luis
sudo useradd -g psychiatry alexandra
id luis
id alexandra
sudo groupadd busadmin
sudo useradd -g busadmin emil
id emil
sudo groupadd collaboration
sudo usermod -aG collaboration luis
sudo usermod -aG collaboration maria
id luis
id maria
```
Reference: [How to Add User to Group in Linux](https://linuxize.com/post/how-to-add-user-to-group-in-linux/)

Hit `useradd -m <username>` to add a new user (the `-m` oprtion creates the home directory it doesn't already exist). Hit `passwd <username>` to add a password to that user. Similarly, user `usermod` and `userdel` to modify and delete users, respectively.

> You can set the user password to expire at first login (i.e., get the user to change its password) by typing `chage -d 0 <user>`
> Use the option `-U` in `usermod` to **unlock**, i.e., remove the password lock.

The command `groupadd` adds a new group. Group info can be found in `/etc/group`. For adding a person to a group, use:
```sh
usermod -aG <group> <user>
```
the `-a` stands for **append** and `G` stands for **group**.

For group and user information use:
```sh
cat /etec/group
id <user>
groups <user>
```