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
One can use wildcards for the _AllowUsers_ line on the `/etc/ssh/sshd_config` file:
```sh
AllowUsers *@192.168.56.100
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

>The order is important because each module depends on the previous module on the stack. 

Understanding the config directives:
- __auth required pam_listfile.so:__ name of module required while authenticating users
- __item=user:__ check the username
- __sense=deny:__ deny user if existing in specified file
- __file=/etc/login/deny:__ name of file which contains the list of user (one user per line)
- __onerr=succeed:__ if an error is encountered PAM will return status PAM_SUCCESS

Reference: [Controlling Root Access](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/security_guide/sec-controlling_root_access)

To check if a program is "PAM-aware" or not, use the `ldd` (List Dynamic Dependancies) command. Examples:
```sh
ldd /bin/su
ldd /usr/sbin/crond | grep libpam.so
```
Added some lines to `/etc/pam.d/login` to restrict access:
```sh
auth	   required     pam_succeed_if.so user ingroup wheel
auth       required     pam_succeed_if.so uid >= 1000
```
Reference: [Configure and use Linux-PAM](https://likegeeks.com/linux-pam-easy-guide/)

### Configuring password using PAM
The `pam_pwquality` module is used to check a password's strength against a set of rules. Add the following line to the password stack in the `/etc/pam.d/passwd` file:
```sh
password   required    pam_pwquality.so retry=2
```
Then edit the file `/etc/security/pwquality.conf` to improve quality as desired.

### Set limits on system resources
Edit the file `/etc/security/limits.conf`. This configuration is implemented via the `pam_limits.so` module which is called for various services configured in `/etc/pam.d/` like this:
```sh
session     required    pam_limits.so
```

Reference: [Desktop Security](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/security_guide/chap-hardening_your_system_with_tools_and_services)

# IPTables
Conceptually iptables is based around the concepts of rules and chains. A rule is a small piece of logic for matching packets. A chain is a series of rules that each packet is checked against, in order. Packets eventually end up in one of a predefined set of targets, which determine what is done with the packet. Each rule defines where to “jump” if a packet matches; this can be to another chain or a target. If the rule does not match, then the packet is checked against the next rule in the chain. Each chain has a policy, which is the target packets jump to if they reach the end of the chain without matching any rules.

Download and start the service (note that `firewalld` will stop):
```sh
sudo yum install iptables iptables-services
sudo systemctl start iptables 
```
Tables (filter, nat, raw, mangle, security) are specified with the `-t` option. Some chains (of rules) are:
- The INPUT chain handles packets that are destined to the local system.
- The OUTPUT chain is for packets that are locally generated.
- The FORWARD chain is used for packets that are routed through the system.

Rules are defined as a set of __matches__ and a __target__. To list rules (in the filter table):
```sh
sudo iptables -L
```
Special values of targets:
- ACCEPT: allows the packet to continue without further checking;
- DROP: refuse access for that packet without sending a response;
- RETURN: returns the packet to the next rule in the previous chain, if the RETURN target is reached within a built-in chain the packet is handled by the chain policy.

Set a policy to accept packets by default:
```sh
iptables -t filter -P INPUT ACCEPT
```
Set a rule (`-A` option to add and `-D` to delete) that drops every packet from the source to detination ip addresses. Since no protocol is specified it will assume all by default:
```sh
iptables -A INPUT -s 192.168.57.100 -d 192.168.57.110 -j DROP
```
Block everything outside the network 192.168.56.0/24 by using the `!` operator:
```sh
iptables -A INPUT ! -s 192.168.57.0/24 -d 192.168.57.110 -j DROP
```
To allow incoming and outgoing tcp packets on a range of ports on the interface enp0s8:
```sh
iptables -A INPUT -p tcp -i enp0s8 --dport 22:25 -j ACCEPT
iptables -A OUTPUT -p tcp -o enp0s8 --sport 22:25 -j ACCEPT
```
To drop incoming request on port 22 from source IP in the 192.168.57.101-192.168.57.199 range only:
```sh
iptables -A INPUT -p tcp --dport 22 -m iprange --src-range 192.168.57.101-192.168.57.199 -j DROP  
```
To drop ssh the out of subnet 192.168.56.0/24:
```sh
sudo iptables -A INPUT -p tcp ! -s 192.168.56.0/24 --dport 22 -j DROP
```

__Syntax:__
`-m iprange -src-range IP-IP -j ACTION`
`-m iprange -dst-range IP-IP -j ACTION`

The `limit` module enables rate limiting against all packets which hit a rule:
```sh
sudo iptables --flush  # start again
sudo iptables --new-chain RATE_LIMIT
sudo iptables -A INPUT -p icmp -j RATE_LIMIT
sudo iptables -A RATE_LIMIT --match limit --limit 50/sec --limit-burst 20 -j ACCEPT
sudo iptables -A RATE_LIMIT -j DROP
```
To delete a rule: 
```sh
sudo iptables -L --line-numbers
sudo iptables -D INPUT 1 # delete the first line of INPUT chain
```
By default rules are stored in `/etc/sysconfig/iptables`, those are the rules that are loaded once the service is started or reloaded. Save with:
```sh
iptables-save > /etc/sysconfig/iptables
```
Or use the command:
```sh
sudo iptables-save
```

References: [Introduction to IPTables](https://linuxacademy.com/guide/15473-introduction-to-iptables/), [How to specify a range of IP addresses or ports ](https://www.cyberciti.biz/tips/linux-iptables-how-to-specify-a-range-of-ip-addresses-or-ports.html), [How To List and Delete Iptables Firewall Rules](https://www.digitalocean.com/community/tutorials/how-to-list-and-delete-iptables-firewall-rules)
This is a VERY good read: [Per-IP rate limiting with iptables](https://making.pusher.com/per-ip-rate-limiting-with-iptables/#fnref:not-idempotent)

### Port forwarding using IPTables
Port forwarding also called _port mapping_ commonly refers to the network address translator gateway changing the destination address and/or port of the packet to reach a host within a masqueraded, typically private, network. Assume interface `enp0s3` is connected to the internet and `enp0s8` is the host-only adapter. As an example, to access a webserver on local network from the internet, use Network Address Translation (NAT):
```sh
sudo iptables -t nat -A PREROUTING -p tcp -d x.x.x.x --dport 8080 -j DNAT --to y.y.y.y:8080
sudo iptables -A FORWARD -p tcp -d y.y.y.y --dport 8080 -j ACCEPT
```
IP x.x.x.x is the external address while y.y.y.y is the internal one running the webserver. Enable forwarding:
```sh
shsysctl net.ipv4.ip_forward=1 
```

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
Services are simply collections of ports with an associated name and description. Using services is easier to administer than ports. To create a service, copy an existing script from `/usr/lib/firewalld/services` to `/etc/firewalld/services`. Reload the firewall:
```sh
sudo firewall-cmd --reload
```
> The predefined zones will probably be more than enough, but it is possible to define new zones just in case.

References: [How To Set Up a Firewall Using FirewallD on CentOS 7](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-using-firewalld-on-centos-7), [Port Forwarding](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/security_guide/sec-port_forwarding#sec-Adding_a_Port_to_Redirect)

### Restrict SSH access to IP
Using _firewalld_:
```sh
sudo firewall-cmd --add-rich-rule 'rule family="ipv4" service name="ssh" source address="192.168.56.100" accept' --permanent
```
Using TCP wrappers: edit the files `/etc/hosts.deny` and `/etc/hosts.allow`.
> Rules in `hosts.allow` take precedence over rules specified in `hosts.deny`.

Using SSH daemon configuration: add desired authentication methods after a `Match Address` in `sshd_config`. See: [Limit SSH access to specific clients by IP address](https://unix.stackexchange.com/questions/406245/limit-ssh-access-to-specific-clients-by-ip-address)

Block connections on ports 22 and 80 to your machine from a machine with a local IP address 192.168.57.100:
```ssh
sudo firewall-cmd --zone=internal --add-rich-rule 'rule family="ipv4" source address="192.168.57.100" port port=22 protocol=tcp reject'
sudo firewall-cmd --zone=internal --add-rich-rule 'rule family="ipv4" source address="192.168.57.100" port port=80 protocol=udp reject'
```
> To set a range of IPs: [IP Sets Using Firewalld](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/security_guide/sec-setting_and_controlling_ip_sets_using_firewalld)

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