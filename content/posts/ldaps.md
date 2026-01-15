+++
title = "How to setup LDAPS with SHA256 Password Hashes"
date = "2023-05-07T17:12:56"
slug = "ldaps"
draft = false
+++

I have been fascinated recently by using ldap to help increase my password strength. I have most of my passwords in my password manager but I would really like to just have one password for doing super admin tasks in my environment so I can skip the password manager step. Also easy password changes without replicating the change to all my servers was something I wanted to do.

I chose against doing this the easy way and just using AD manage the ldap server. After four or five days struggling I do see why people us AD just for the convince and I probably would have done it if I was not already down a rabbit hole.

I am going to use this as a guide on how to setup an openldap LDAPS server that also uses SHA256 password hashes. I found that regular openldap natively uses very insecure hashes. You have to dynamically load a module which I personally don't like. I think you can compile openldap your self to already have the module loaded but I would rather just load it. Apparently this is not going to be fixed to be native   
<https://www.openldap.org/lists/openldap-bugs/201205/msg00055.html>  
We kind of are just relying on the Atlassian to do it <https://bugs.openldap.org/show_bug.cgi?id=5660>

 I will note my setup may be broken as user filters do not work at all. This is terrible but I am not an expert and don't feel the hours spent investigating will give me much value. If you are an expert and happen apon this I would be indebted to you if you let me know.

Setup
-----

1. **Install and configure the base openldap database**


```
sudo apt update
sudo apt upgrade
sudo su
apt install slapd ldap-utils
dpkg-reconfigure slapd
```
Then for the questions you want to answer  
Omit OpenLDAP server configuration?: No  
DNS Domain name: Your ldap domain name. Should not be the fqdn of your host Ex: phantasmfour.com(going to user phantasmfour as an example everywhere but sub with yours)  
Org name: first part of domain name: EX: phantasmfour  
Then enter your same admin password  
Remove DB when purged: Yes  
Move Old Files: Yes

Then you can run `slapcat` and you should see that base database of your whole org. 

**2. Create OU and Add Users**

Create a baseFile `nano baseFile`  
then enter the base OUs you want to make. These are standard


```
dn: ou=people,dc=phantasmfour,dc=com
objectClass: organizationalUnit
ou: people

dn: ou=groups,dc=phantasmfour,dc=com
objectClass: organizationalUnit
ou: groups
```
Then run `ldapadd -x -D "cn=admin,dc=phantasmfour,dc=com" -W -f baseFile`  
This will add these OUs in.

Now you can create a base file for importing users, but first you need to generate the hash for your users password.  
Run `slappasswd -h '{SHA256}' -o module-path=/usr/lib/ldap -o module-load=pw-sha2`  
Enter the password you want to create and paste your hash into a notepad somewhere.   
Now we can create the userImport file `nano userImport`


```
dn: uid=<username>,ou=people,dc=phantasmfour,dc=com
objectClass: top
objectClass: posixAccount
objectClass: inetOrgPerson
objectClass: organizationalPerson
cn: <full name>
sn: <last name>
givenName: <first name>
uid: <username>
uidNumber: <uid_number>
gidNumber: <gid_number>
homeDirectory: /home/<username>
userPassword: {SHA256}<encrypted_password>
```
Then you add the user by running `ldapadd -x -D "cn=admin,dc=phantasmfour,dc=com" -W -f userImport`. You can add as many users as you want like this.   
You can check the users in the group OU by running   
`ldapsearch -x -b "ou=people,dc=phantasmfour,dc=com`

**3. Generate Cert and key files**

Now you need to generate cert and key files in an acceptable format for openldap. I found a great guide for this I will link [here](https://www.skynats.com/server-management/install-and-configure-open-ldap-server-on-ubuntu-20-04/). The one mistake here he made was that the ldap\_ssl.ldif file needs -'s. It should look like this  



```
dn: cn=config
changetype: modify
add: olcTLSCACertificateFile
olcTLSCACertificateFile: /etc/ldap/sasl2/ca-certificates.crt
-
add: olcTLSCertificateFile
olcTLSCertificateFile: /etc/ldap/sasl2/ldap_server.crt
-
add: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/ldap/sasl2/ldap_server.key
```
Make sure openldap can read those key/cert files  
Here you should run `less /etc/ldap/slapd.d/cn\=config.ldif`  
This should show the cert files added. Most sites mention this as the slapd.conf file which has moved to this.

You then need to edit slapd to be running on ldaps `nano /etc/default/slapd` to have ldaps in the services line like this  
`SLAPD_SERVICES="ldap:/// ldapi:/// ldaps:///"`  
Run a `systemctl restart slapd`

If you want to test your ldap auth from an ldap linux client you need to make sure TLS\_REQCERT is set to demand or never to make sure you can at least auth in your /etc/ldap/ldap.conf file. You should do this on the local server at least so you can test your configs

**4. Create LDAP Groups**

Make a groupMake file with contents like this


```
dn: cn=sudousers,ou=groups,dc=phantasmfour,dc=com
objectClass: top
objectClass: groupOfNames
cn: sudousers
member: uid=user1,ou=people,dc=phantasmfour,dc=com
```
Then just run a `ldapadd -x -D cn=admin,dc=phantasmfour,dc=com -W -H ldaps://localhost -f groupMake`

You can add a user to an existing group with something like this


```
dn: cn=sudousers,ou=groups,dc=phantasmfour,dc=com
changetype: modify
add: member
member: uid=user2,ou=people,dc=phantasmfour,dc=com
```
Then just run a `ldapadd -x -D cn=admin,dc=phantasmfour,dc=com -W -H ldaps://localhost -f groupAdd`

**5. Enable the pw-sha2 module**

Make a moduleLoad file with contents like this


```
dn: cn=module{0},cn=config
changetype: modify
add: olcModuleLoad
olcModuleLoad: pw-sha2
```
Then run a `ldapadd -Y EXTERNAL -H ldapi:/// -f moduleLoad`

At this point you need to pull the full config to just check it over and make sure you see the module loaded. `ldapsearch -LLL -Q -Y EXTERNAL -H ldapi:/// -b cn=config > slapd.conf.ldif`

Restart slapd `systemctl restart slapd`

Test authentication from another linux system with something like this `ldapsearch -H ldap://ldap.phantasmfour.com -D "uid=user1(USERNAME TO SIGN IN AS),ou=people,dc=phantasmfour,dc=com" -W -x -b "dc=phantasmfour,dc=com" "(objectClass=*)"`

**6. Change admin password**

I cannot confirm but the admin password is probably already sha1 so lets change it to using a sha256 hash

Generate your sha256 hash ``slappasswd -h '{SHA256}' -o module-path=/usr/lib/ldap -o module-load=pw-sha2``   
Create a file adminPassword  



```
dn: olcDatabase={0}config,cn=config
changetype: modify
replace: olcRootPW
olcRootPW: {SHA256}fjkshgjakfds
```
Then run `ldapmodify -Y EXTERNAL -H ldapi:/// -f adminPassword`

This should change the admin password.

If you want to change your user password you can create a file like


```
dn: uid=user1,ou=people,dc=phantasmfour,dc=com
changetype: modify
replace: userPassword
userPassword: {SHA256}cHpjkfghdfjk
```
And run `ldapmodify -x -D cn=admin,dc=phantasmfour,dc=com -W -H ldaps://localhost -f newPassword`

Conclusion
----------

I am not an openldap expert. I have this to the point that it is working and is generally easy to manage.  If the hashes are actually being stored as SHA256 somewhere I would like to explore slapcat shoes a sha1 hash but I am guessing it is a sha256 hash hashed again as sha1. I am looking to validate this somehow but SSHA does add a salt value. I beleive Windows stores ldap hashes in NT hash so I at least feel better about doing it this way.

My config for some reason did not let me use user Filters at all. Why I cannot do this is confusing and caused some issues with my proxmox config. I ended up just having to import all users and since proxmox requires you to have bother users and groups imported it was not too big of a deal.

One thing I never got working was having openldap work as an authentication mechanism for SSH. I could not find a comprehensive guide of this working with openldap so if I figure it out I may add it. 

There are a lot of fun things I did not explore with openldap. You can add in a gui component to manage everything. You can also store a lot of different things in ldap from what I am reading. If this was an enterprise environment I would probably shut down my ldap port or just force startls but that was not done here.  
At the point where I am at I have my data encrypted at rest and in transit via an ldap server so I am happy.

Sources:  
<https://kifarunix.com/install-and-setup-openldap-server-on-ubuntu-20-04/>

<https://www.skynats.com/server-management/install-and-configure-open-ldap-server-on-ubuntu-20-04/>

<https://web.archive.org/web/20130306011040/http://rogermoffatt.com/2011/08/24/ubuntu-openldap-with-ssltls/>

<https://computingforgeeks.com/how-to-configure-ubuntu-as-ldap-client/>

<https://computingforgeeks.com/install-and-configure-ldap-account-manager-on-ubuntu/>

<https://www.vennedey.net/resources/0-Getting-started-with-OpenLDAP-on-Debian>

<https://openldap-technical.openldap.narkive.com/WqXgmHim/passwords-hashing-and-binds>

