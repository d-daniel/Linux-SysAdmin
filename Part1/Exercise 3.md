# Exercise 3

Created the *surgery* project directory and use command `chgrp` to change group ownership. Option `-R` enables recursive operations into subdirectories:
```sh
cd /
sudo mkdir -p surgery/project
sudo chgrp -R surgery /surgery/project/
sudo chmod 2770 /surgery/project/ 
```
Same thing for the *psychiatry* group:
```sh
sudo mkdir -p psychiatry/project
sudo chgrp -R psychiatry /psychiatry/project/
sudo chmod 2770 /psychiatry/project/ 
```

Explaining the permissions `2770` in the `chmod` command above:
`2` turns on the setGID bit, implying–newly created subfiles inherit the same group as the directory, and newly created subdirectories inherit the set GID bit of the parent directory (__important:__ do not use `2` because it's not POSIX compliant, use `g+s`)
`7` gives rwx permissions for owner
`7` gives rwx permissions for group
`0` gives no permissions for others

### The *collaboration* group
```sh
sudo mkdir /collaboration
sudo chgrp -R collaboration collaboration/
sudo chmod 2770 collaboration/
sudo setfacl -d -m u:emil:r-x collaboration/
getfacl collaboration/
```
___
Get only the user and group with `stat`: 
```sh
stat -c "%U %G" /path/to/file
```
___

If you need to do something in place of a user, use:
```sh
sudo -u<user> <command>
sudo -u henry ls -la /surgery/project
```