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
192.168.56.147  ldapserver  ldapserver.training.edu 
```
Then, set the hostname:
```sh
sudo hostnamectl set-hostname ldapServer.training.edu
```
Logout and login to see the changes.
If you want to SSH from your computer, edit `~/.ssh/config`:
```sh
Host ldapserver
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
sudo systemctl status slapd
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

Once installed, we have to generate a password for the LDAP user with command `slappasswd` (consider using SHA-512). Copy the password to the clipboard. Choose a suffix. A suffix is the root of your directory tree (`olcSuffix`). Before adding entries, configure a database (visit a website for a sample configuration file). Edit an LDIF file and paste the password in `oclRootPW`. My `db.ldif` file looks as follows (note the `olcRootDN`):
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
Replace `olcAccess` attribute to allow access to the LDAP database to the `ldapadm` user specified before (`cn=ldapadm,dc=training,dc=edu`). Another option is to set `olcAccess` to an organizational unit of service accounts. My `monitor.ldif` file is like this:
```sh
dn: olcDatabase={1}monitor,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to * 
            by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external, cn=auth" read 
            by dn.base="cn=ldapadm,dc=training,dc=edu" read 
            by * none
```
This is just the `cn=monitor` subtree; I'll experiment with ACL for the databese itself later. Use `ldapmodify` once more:
```sh
sudo ldapmodify -H ldapi:/// -f monitor.ldif 
```
Check the suitability of the OpenLDAP slapd configuration with `sudo slaptest -v`. It's going to complain that there's no DB_CONFIG file found in directory `/var/lib/ldap`. Copy the sample database and change the ownership (in CentOS 7, the owner is `ldap:ldap`):
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

dn: ou=groups,dc=training,dc=edu
objectClass: organizationalUnit
ou: groups

dn: ou=system,dc=training,dc=edu
objectClass: organizationalUnit
ou: system

dn: ou=users,dc=training,dc=edu
objectClass: organizationalUnit
ou: users
```

Hit `sudo ldapadd -W -D "cn=ldapadm,dc=training,dc=edu" -f entry.ldif`. Note that `-W` is to prompt for authentication and `-D` is to use `binddn` to bind to the LDAP directory. Create a user `s191529` in `users` with `sudo ldapadd -W -D "cn=ldapadm,dc=training,dc=edu" -f s191529.ldif`. File `s191529.ldif` looks like this:
```sh
dn: uid=s191529,ou=users,dc=training,dc=edu
objectClass: top
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
cn: s191529
uid: s191529
uidNumber: 9999
gidNumber: 9999
homeDirectory: /home/s191529
loginShell: /bin/bash
userPassword: {crypt}x
shadowLastChange: 17058
shadowMin: 0
shadowMax: 99999
shadowWarning: 7
```
Set a password for user `s191529` with:
```sh
sudo ldappasswd -s <password> -W -D "cn=ldapadm,dc=training,dc=edu" "cn=s191529,ou=users,dc=training,dc=edu"
```
Check the user exists with:
```sh
sudo ldapsearch -x cn=s191529 -b dc=training,dc=edu
```
Delete the user if necessary:
```sh
sudo ldapdelete -W -D "cn=ldapadm,dc=training,dc=edu" "cn=s191529,ou=users,dc=training,dc=edu"
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
Create a __self-signed certificate__ for LDAP server. Generate both certificate and private key in `/etc/openldap/certs/ directory`. 
A step-by-step guide is here: [Configure OpenLDAP with SSL on CentOS 7 / RHEL 7](https://www.itzgeek.com/how-tos/linux/centos-how-tos/configure-openldap-with-ssl-on-centos-7-rhel-7.html)

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

It might be useful to skip the certificate validation in a TLS session (custom CA-signed certificate was a headache). Add a line to file `/etc/openldap/ldap.conf`:
```sh
TLS_REQCERT allow
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
Install required packages (if not installed yet):
```sh
yum install -y openldap-clients nss-pam-ldapd
```
Execute the `authconfig` command: 
```sh
sudo authconfig --enableldap --enableldapauth --ldapserver=ldaps://ldapserver.training.edu --ldapbasedn="dc=training,dc=edu" --enablemkhomedir --update
```
Edit the `nslcd.conf` file. The below setting will disable the certificate validation done by clients as we are using a self-signed certificate:
```sh
TLS_REQCERT allow
```
Restart the server and then verify:
```sh
sudo systemctl restart nslcd
sudo getent passwd <user> 
```

> This method of encrypting LDAP connections is actually deprecated and the use of STARTTLS encryption is recommended instead. 

# Apache Directory Studio LDAP Browser (GUI)
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
First and most important: learn how to use `ldapsearch`. Second, to truly understand the `cn=config`, go to `/etc/openldap/slapd.d` and see the contents of `config.ldif`. Check the attribute __olcAccess__ includes the root user (the one who can actually change the `cn=config`). Check the ACLs of the database:    
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

> IMPORTANT: when setting ACLs, always place the most specific rules first, and only then the generic rules. LDAP is extremely sensible with respect to formatting and syntax when it comes to updating `olcAccess`.

# Adding samba schema to the LDAP Server
Download the tools (`smbldap-tools`) that contain the LDIF file and load the schema on the LDAP server:
```sh
sudo yum install samba
sudo ldapadd -H ldapi:/// -f /usr/share/doc/samba-4.9.1/LDAP/samba.ldif
sudo ldapsearch -H ldapi:/// -b cn=schema,cn=config "(objectClass=olcSchemaConfig)" dn
```
You dont't need `smbldap-tools` anymore; remove it from the LDAP server machine. On the Samba server (IP = 192.168.56.101), add LDAP parameters to `/etc/samba/smb.conf`:
```
[global]
        workgroup = SAMBA
        netbios name = CENTOS
        server string = CentOS Samba Server
        wins support = yes
        name resolve order = bcast wins lmhosts

        passdb backend = ldapsam:ldaps://192.168.56.147
        log level = 3
        log file = /var/log/samba/log.%m
        max log size = 50
        ldap admin dn = cn=overlord,ou=system,dc=training,dc=edu
        ldap group suffix = ou=groups
        ldap passwd sync = Yes
        ldap suffix = dc=training,dc=edu
        ldap user suffix = ou=users
        ldap ssl = off

[training]
        comment=shared directories for training
        browsable=yes
        path=/new_home
        public=no
        writable=yes
        create mask=0770
        Force create mode=0770
```
Run `sudo smbpasswd -W` to let samba know the password for the admin dn. Check `smb` to see if you can connect to the LDAP server. If sucessful, notice that `DN: sambaDomainName=CENTOS,dc=training,dc=edu` is added to LDAP database (notice attribute SID). Migrate LDAP users that you want to include in LDAP-backed Samba with `smbpasswd`:
```sh
sudo smbpasswd -a username
```
In order for this to work, you have to add the user to the samba server as well. Chech the LDAP database to verify that samba-related attributes are now added to that user (SID, for example). Test connection on a Windows machine.

> IMPORTANT: on every machine, set __TLS_REQCERT allow__ in `ldap.conf` file to skip certificate validation.

Reference: [Samba and LDAP](https://help.ubuntu.com/lts/serverguide/samba-ldap.html)

# Configure LDAP Sync Replication
Create a user who will have a read access to all LDAP objects in the __master server__. My LDIF file looks like this:
```sh
dn: cn=rpluser,dc=training,dc=edu
objectClass: simpleSecurityObject
objectClass: organizationalRole
cn: rpluser
description: LDAP server replicator
userPassword: rpluser
```
Add the user to the server:
```sh
sudo ldapadd -x -W -D cn=ldapadm,dc=training,dc=edu -v -f create_repl_user.ldif
```
Enable `syncprov`. My `enable_sync_prov.ldif` looks like this:
```sh
dn: olcDatabase={2}hdb,cn=config
changetype: modify
delete: olcAccess
olcAccess: {0}to attrs=userPassword by dn.subtree="ou=system,dc=training,dc=edu" write by self write by anonymous auth by * none
olcAccess: {1}to * by dn.base="cn=overlord,ou=system,dc=training,dc=edu" manage by self write by * read
-
add: olcAccess
olcAccess: {0}to attrs=userPassword by dn.subtree="ou=system,dc=training,dc=edu" write by self write by dn.base="cn=rpluser,dc=training,dc=edu" read by anonymous auth by * none
olcAccess: {1}to * by dn.base="cn=overlord,ou=system,dc=training,dc=edu" manage by self write by * read

dn: cn=module,cn=config
changetype: add
objectClass: olcModuleList
cn: module
olcModulePath: /usr/lib64/openldap
olcModuleLoad: syncprov.la

dn: olcOverlay=syncprov,olcDatabase={2}hdb,cn=config
changetype: add
objectClass: olcOverlayConfig
objectClass: olcSyncProvConfig
olcOverlay: syncprov
olcSpCheckpoint: 100 10
olcSpSessionlog: 100
```
Update the configuration on LDAP master server:
```sh
sudo ldapadd -Y EXTERNAL -H ldapi:/// -W -D cn=ldapadm,dc=training,dc=edu -v -f enable_sync_prov.ldif
sudo ldapsearch -H ldapi:/// -b cn=config dn
```

Set up the slave server. I had to download LDAP, configure the database (`rootDN`, `oclRootPW`, `olcSuffix`, LDAP over SSL, etc) and add the schema files. Configure the replication (`enable_sync_consumer.ldif`):
```sh
dn: olcDatabase={2}hdb,cn=config
changetype: modify
add: olcSyncRepl
olcSyncRepl: rid=147
  provider="ldaps://192.168.56.147:636/"
  bindmethod=simple
  binddn="cn=rpluser,dc=training,dc=edu"
  credentials=rpluser
  searchbase="dc=training,dc=edu"
  scope=sub
  schemachecking=on
  type=refreshAndPersist
  retry="30 5 300 3"
  interval=00:00:05:00
  starttls=no
  tls_reqcert=allow
```
Upload the configuration:
```sh
sudo ldapmodify -H ldapi:/// -f enable_sync_consumer.ldif
```
Restart `slapd` on the slave server. Those should work:
```sh
sudo ldapsearch -H ldaps:/// -D "cn=rpluser,dc=training,dc=edu" -W cn=overlord -b ou=system,dc=training,dc=edu
sudo ldapsearch -H ldaps:/// -D "cn=rpluser,dc=training,dc=edu" -W cn=s191529 -b ou=users,dc=training,dc=edu
```
Reference: [How to configure OpenLDAP Master-Slave Replication](https://www.itzgeek.com/how-tos/linux/configure-openldap-master-slave-replication.html) 

### Multi-Master Replication
Multi-Master replication is a replication technique using Syncrepl to replicate data to multiple provider ("Master") Directory servers.
__Pros:__
1. If any provider fails, other providers will continue to accept updates;
2. Avoids a single point of failure;
3. Providers can be located in several physical sites, i.e., distributed across the network/globe;
4. High availability.

__Cons:__
1. Breaks the data consistency guarantees of the directory model;
2. If connectivity with a provider is lost because of a network partition, then "automatic failover" can just compound the problem;
3. Typically, a particular machine cannot distinguish between losing contact with a peer because that peer crashed, or because the network link has failed;
4. If a network is partitioned and multiple clients start writing to each of the "masters" then reconciliation will be a pain.

For further details, see: http://www.openldap.org/doc/admin24/replication.html#N-Way%20Multi-Master%20replication

# Appendix
Useful documentation: [How To Configure OpenLDAP and Perform Administrative LDAP Tasks](https://www.digitalocean.com/community/tutorials/how-to-configure-openldap-and-perform-administrative-ldap-tasks)
Useful step-by-step guide: [Step by Step OpenLDAP Server Configuration on CentOS 7/RHEL 7](https://www.itzgeek.com/how-tos/linux/centos-how-tos/step-step-openldap-server-configuration-centos-7-rhel-7.html/2)

> DO NOT edit the LDIF configuration file directly!
> `-H ldapi:///` - use UNIX-domain socket (`/var/run/ldapi`)
> `-Y EXTERNAL` - use EXTERNAL mechanism for SASL (Simple Authentication and Security Layer)
___
More references: [Configuring OpenLDAP for Linux Authentication](https://tylersguides.com/guides/configuring-openldap-for-linux-authentication/), [LDAP Concepts & Overview](http://www.zytrax.com/books/ldap/ch2/index.html#overview)
