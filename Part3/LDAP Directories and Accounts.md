# Lightweight Directory Access Protocol (LDAP)
LDAP is a client-server arrangement for storing information (this includes access permissions). LDAP sits on top of TCP/IP. The standard TCP ports for LDAP are 389 for unencrypted communication and 636 for LDAP over a TLS-encrypted channel (TLS stands for Transport Layer Security). LDAP can be seen as centralized user account management, that is, a common database in which to keep all information related to user accounts. LDAP keeps a central database in which users, computers, and network objects are registered. OpenLDAP is the open source implementation of LDAP that runs on Linux/UNIX systems. LDAP is characterized as a write-once-read-many-times service.

### LDAP Terminology
- __Directory__: database that stores information about objects
- __Entry__: a single unit in an LDAP directory
- __Attribute__: detail of an entry
- __Matching rule__: search criteria for matching entries
- __Object class__: structure of required and optional attributes for an entry
- __Schema__: attributes, object classes and matching rules packaged
- __LDIF (Lightweight Directory Interchange Format)__: plaintext representation of an LDAP entry
- __DN (Distinguished Name)__: unique id for entry
- __RDN (Relative Distinguished Name)__: unique id for components of an entry

### LDAP Attribute Details
- __Attribute type__: describes the information
- __Attribute value__: the data itself

### LDAP Common Attribute Types
1. Common name (cn)
2. Domain component (dc)
3. Country
4. Mail (mail)
5. Address (streetAddress)
6. Organization (o)
7. Organizational unit (ou)
8. Surname (sn)
9. Telephone number (telephoneNumber)

### Set hostname (optional)
My LDAP server has IP 192.168.56.147 (see `/etc/sysconfig/network-scripts/ifcfg-enp0s3`). Hit `sudo vim /etc/hosts` and add the following line:
```sh
192.168.56.147  ldapServer  ldapServer.training.edu 
```
Then, set the hostname:
```sh
sudo hostnamectl set-hostname ldapServer.training.edu
```
Logout and login to see the changes.
If you want to SSH from your computer, edit `~/.ssh/config`:
```sh
Host ldapServer
     HostName 192.168.56.147
     User daniela
     IdentityFile ~/.ssh/id_rsa
```
___

# Set up an LDAP Server
Install packages and check the service:
```sh
sudo yum install openldap-clients openldap-servers
sudo systemctl start slapd
sudo systemctl ststus slapd
sudo systemctl enable slapd
```
> OpenLDAP naming convention: server commands start with __slap__ and client commands start with __ldap__.

Set SE boolean (option `-P` for persistency):
```sh
sudo setsebool -P authlogin_nsswitch_use_ldap=1
getsebool -a | less
sudo ss -lntu | grep 389
```
> Command `ss` is an utility to investigate sockets.

Once installed, we have to generate a password for the LDAP user with command `slappasswd` (consider using SHA-512). Copy the password to the clipboard. Choose a suffix. A suffix is the root of your directory tree (`olcSuffix`). Before adding entries, configure a database (visit the website for a sample configuration file). Edit an LDIF file and paste the password in `oclRootPW`. Mine looks as follows (note the `olcRootDN`):
```sh
dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=training,dc=edu

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: cn=ldapadm,dc=training,dc=edu

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootPW
olcRootPW: {SSHA}xG7x63R1gt+d5ehMW+NtdULh7Bd3+sqT
```
> IMPORTANT: use `{3}mdb` or `{2}hdb`, for correctly identifying backends. Note that __hdb__ will soon be deprecated in favor of the new __mdb__ backend.

Next, load the file with `ldapmodify` command:
```sh
sudo ldapmodify -H ldapi:/// -f db.ldif
```
Replace `olcAccess` attribute to allow access to the LDAP database to the `ldapadm` user specified before (`cn=ldapadm,dc=training,dc=edu`). Another option is to set `olcAccess` to an OU of service accounts. My LDIF file (`monitor.ldif`) is like this:
```sh
dn: olcDatabase={1}monitor,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to * 
            by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external, cn=auth" read 
            by dn.base="cn=ldapadm,dc=training,dc=edu" read 
            by * none
```
This is just the `cn=Monitor` subtree; experiment with ACL later. Use `ldapmodify` once more:
```sh
sudo ldapmodify -H ldapi:/// -f monitor.ldif 
```
Check the suitability of the OpenLDAP slapd configuration WITH `sudo slaptest -v`. It's going to complain that there's no DB_CONFIG file found in directory `/var/lib/ldap`. Copy the sample database and change the ownership (in CentOS 7, the owner is `ldap:ldap`):
```sh
sudo cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
sudo chown -R ldap:ldap /var/lib/ldap/
```
Add some schema files:
```sh
ls -l /etc/openldap/schema/
sudo ldapadd -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
sudo ldapadd -H ldapi:/// -f /etc/openldap/schema/nis.ldif
sudo ldapadd -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
```
Typically, you organize directory entries into organizational units to make them easier to find and manage. Manually create an entry for `dc=training,dc=edu` in our LDAP server. The easiest way to do this is to create an LDIF file for this entry and pass it to the `ldapadd` command. My file `entry.ldif` is like this:
```sh
dn: dc=training,dc=edu
objectClass: dcObject
objectClass: organization
dc: training
o: training

dn: ou=Bioinformatics,dc=training,dc=edu
objectClass: organizationalUnit
ou: Bioinformatics

dn: ou=users,dc=training,dc=edu
objectClass: organizationalUnit
ou: users
```
Hit `sudo ldapadd -x -W -D "cn=ldapadm,dc=training,dc=edu" -f entry.ldif`. Note that `-x` is not to use SASL, `-W` is to prompt for authentication ans `-D` is to use `binddn` to bind to the LDAP directory. 
Now create a user `s191529` in `users` with `sudo ldapadd -x -W -D "cn=ldapadm,dc=training,dc=edu" -f s191529.ldif`. File `s191529.ldif` looks like this:
```sh
dn: uid=s191529,ou=users,dc=training,dc=edu
objectClass: top
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
cn: s191529
uid: s191529
uidNumber: 9999
gidNumber: 100
homeDirectory: /home/s191529
loginShell: /bin/bash
gecos: CentOS 7 User
userPassword: {crypt}x
shadowLastChange: 17058
shadowMin: 0
shadowMax: 99999
shadowWarning: 7
```
Set a password for user `s191529` with:
```sh
sudo ldappasswd -s <password> -W -D "cn=ldapadm,dc=training,dc=edu" -x "uid=s191529,ou=users,dc=training,dc=edu"
```
Check the user exists with:
```sh
sudo ldapsearch -x cn=s191529 -b dc=training,dc=edu
```
Delete the user if necessary:
```sh
sudo ldapdelete -W -D "cn=ldapadm,dc=training,dc=edu" "uid=s191529,ou=users,dc=training,dc=edu"
```
Firewall:
```sh
sudo firewall-cmd --permanent --add-service=ldap
sudo firewall-cmd --reload
```

# Set up LDAP Client 
Install the packages (PAM stands for Pluggable Authentication Module):
```sh
sudo yum install openldap-clients nss-pam-ldapd
sudo authconfig --enableldap --enableldapauth --ldapserver=192.168.56.147 --ldapbasedn="dc=training,dc=edu" --enablemkhomedir --update
sudo systemctl restart nslcd
sudo systemctl status nslcd
sudo systemctl enable nslcd
```
Verify connection:
```sh
sudo getent passwd s191529
```
Or SSH the server as user `s191529`. It will work! =D

# Configuring LDAP over SSL (Server)
Install required packages (if not already installed):
```sh
sudo yum install -y openldap-clients nss-pam-ldapd
```
Create a self-signed certificate for LDAP server. Generate both certificate and private key in `/etc/openldap/certs/ directory`. The step-by-step guide is here: [Configure OpenLDAP with SSL on CentOS 7 / RHEL 7](https://www.itzgeek.com/how-tos/linux/centos-how-tos/configure-openldap-with-ssl-on-centos-7-rhel-7.html)

Change ownership:
```sh
chown -R ldap:ldap /etc/openldap/certs/
```
Check default values of TLS related attributes:
```sh
sudo slapcat -b "cn=config" | egrep "olcTLSCertificateFile|olcTLSCertificateKeyFile"
```
Replace these values using an LDIF file (in my case `certs.ldif`):
```sh
dn: cn=config
changetype: modify
replace: olcTLSCertificateFile
olcTLSCertificateFile: /etc/openldap/certs/ldapserver.crt
-
replace: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/openldap/certs/ldapserver.key
```
Use `ldapmodify`:
```sh
sudo ldapmodify -H ldapi:/// -f certs.ldif
```
Edit the `/etc/sysconfig/slapd` file and configure OpenLDAP to listen over SSL:
```sh
SLAPD_URLS="ldapi:/// ldap:/// ldaps:///"
```

Restart the __slapd__ service:
```sh
sudo systemctl restart slapd
```
Verify the LDAP service. It should now be listening on TCP port 636 as well:
```sh
sudo ss -lntu | grep 636
```

In the server, we’ll have to allow incoming traffic to port ldaps (636):
```sh
sudo firewall-cmd --permanent --add-service=ldaps
sudo firewall-cmd --reload
```

# Configuring LDAP over SSL (Client)
Install required packages:
```sh
yum install -y openldap-clients nss-pam-ldapd
```
Execute the `authconfig` command: 
```sh
sudo authconfig --enableldap --enableldapauth --ldapserver=ldaps://ldapserver.training.edu --ldapbasedn="dc=training,dc=edu" --enablemkhomedir --update
```
Edit the `nslcd.conf` file. The below setting will disable the certificate validation done by clients as we are using a self-signed certificate:
```sh
tls_reqcert allow
```
Restart the server and then verify:
```sh
sudo systemctl restart nslcd
sudo getent passwd <user> 
```

# Apache Directory Studio LDAP Browser
Download the application from [Apache Directory](https://directory.apache.org/studio/). Extract it and run: `./ApacheDirectoryStudio`. 
Connect Apache Directory Studio to your LDAP server. Click __File > New__ and then select __LDAP Connection__. 
The __Network Parameter__ tab looks as follows:
```sh
Connection name: OpenLDAP Server
Hostname: 192.168.56.147
Port: 636
Encryption method: Use SSL encryption (ldaps://)
```
And the __Authentication__ looks like this:
```sh
Bind DN or user: cn=ldapadm,dc=training,dc=edu
Bind password: <password>
```
And that's it!

# Setting OpenLDAP ACLs
First and most important: learn how to use `ldapsearch`. Second, to truly understand the `c=config`, go to `/etc/openldap/slapd.d` and see the contents of `config.ldif`: check the attribute __olcAccess__ is the root user (the one who can actually change the `cn=config`). Check the ACLs of the database:    
```sh
sudo ldapsearch -H ldapi:/// -b  cn=config 'olcDatabase=hdb'
```
I set the policy as follows (`acls.ldif`):
```sh
dn: olcDatabase={2}hdb,cn=config
changetype: modify
add: olcAccess
olcAccess: {0}to attrs=userPassword by dn.subtree="ou=system,dc=training,dc=edu" write by self write by anonymous auth by * none
olcAccess: {1}to * by dn.base="cn=overlord,ou=system,dc=training,dc=edu" manage by self write by * read
```

> IMPORTANT: when setting ACLs, always place the most specific rules first, and only then the generic rules.

# Adding samba schema to the LDAP Server
Download the tools to have that contain the LDIF file and load the schema:
```sh
sudo yum install smbldap-tools
sudo ldapadd -H ldapi:/// -f /usr/share/doc/samba-4.9.1/LDAP/samba.ldif
sudo ldapsearch -H ldapi:/// -b cn=schema,cn=config "(objectClass=olcSchemaConfig)" dn
```

# Appendix
Useful documentation: [How To Configure OpenLDAP and Perform Administrative LDAP Tasks](https://www.digitalocean.com/community/tutorials/how-to-configure-openldap-and-perform-administrative-ldap-tasks)

> DO NOT edit the LDIF configuration file directly!
> `H ldapi:///` - use UNIX-domain socket (`/var/run/ldapi`)
> `-Y EXTERNAL` - use EXTERNAL mechanism for SASL (Simple Authentication and Security Layer)
___
References: [Configuring OpenLDAP for Linux Authentication](https://tylersguides.com/guides/configuring-openldap-for-linux-authentication/), [LDAP Concepts & Overview](http://www.zytrax.com/books/ldap/ch2/index.html#overview)