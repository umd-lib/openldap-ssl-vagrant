# openldap-ssl-vagrant

This page describes how to configure a Vagrant box to run as an SSL-enabled OpenLDAP server, primarily for use by a Hippo dev environment as an LDAP security provider.

These steps largely follow what is in the Ubuntu manual – see <https://help.ubuntu.com/12.04/serverguide/openldap-server.html>

## Configuration Steps

1) Clone the repository to a convenient directory.  attached "Vagrantfile" to a convenient directory.

2) In the directory containing the "Vagrantfile", run the following command to build the virtual machine:

```
> vagrant up
```

This only does the following:

* Retrieves a 32-bit Ubuntu v12.04 ("Precise Pangolin")
* Creates a private network, with the IP address of the guest being 192.168.33.10

3) Log in to the virtual machine:

```
> vagrant ssh
```

4) Update the packages on the machine:

```
vagrant> sudo apt-get update
```

5) Install the OpenLDAP packages:

```
vagrant> sudo apt-get install slapd ldap-utils
```

You will be prompted for a password to use with OpenLDAP. Type in something like "admin". Once completed, OpenLDAP will be running.

6) Reconfigure the slapd installation:

```
vagrant> sudo dpkg-reconfigure slapd
```

When prompted, select the following answers:

|Prompt                                 | Answer  |
|---------------------------------------|---------|
| Omit OpenLDAP server configuration    | No      |
| DNS domain name                       | umd.edu |
| Organization name                     | umd     |
| Administration password               | admin   |
| Confirm password                      | admin   |
| Database backend to use               | HDB     |
| Database removed when slapd is purged | Yes     |
| Move old database                     | No      |
| Allow LDAPv2 protocol                 | No      |

7) Switch to the "/vagrant" directory, and load the "default.ldif" file into LDAP:

```
> cd /vagrant
> ldapadd -c -D cn=admin,dc=umd,dc=edu -W -f default.ldif
```

You will be prompted for a password, type "admin".

8) Test that the load was successful:

```
vagrant> ldapsearch -x -LLL -b dc=umd,dc=edu 'cn=foo' cn objectClass
```

This will return the following output:

```
dn: uid=foo,ou=people,dc=umd,dc=edu
objectClass: inetOrgPerson
cn: foo

```

9) Turn on more logging:

```
vagrant> sudo ldapmodify -Q -Y EXTERNAL -H ldapi:/// -f logging.ldif
```

This cause OpenLDAP to send more verbose logging to /var/log/syslog.
SSL Configuration

10) Install the packages necessary for generating the SSL certificates:

```
vagrant> sudo apt-get install gnutls-bin ssl-cert
```

11) Generate the private key:

```
vagrant> sudo sh -c "certtool --generate-privkey > /etc/ssl/private/cakey.pem"
```

12) Create the /etc/ssl/ca.info file:

```
vagrant> sudo vi /etc/ssl/ca.info
```

with the following contents:

```
cn = umd.edu
ca
cert_signing_key
```

13) Generate the self-signed CA certificate:

```
vagrant> sudo certtool --generate-self-signed --load-privkey /etc/ssl/private/cakey.pem --template /etc/ssl/ca.info --outfile /etc/ssl/certs/cacert.pem
```

14) Generate the private key for the server:

```
vagrant> sudo certtool --generate-privkey --bits 1024 --outfile /etc/ssl/private/umd_slapd_key.pem
```

15) Create the /etc/ssl/umd.info file:

```
vagrant> sudo vi /etc/ssl/umd.info
```

with the following contents:

```
organization = UMD
cn = umd.edu
tls_www_server
encryption_key
signing_key
expiration_days = 3650
```

16) Create the server certificate:

```
vagrant> sudo certtool --generate-certificate --load-privkey /etc/ssl/private/umd_slapd_key.pem --load-ca-certificate /etc/ssl/certs/cacert.pem --load-ca-privkey /etc/ssl/private/cakey.pem --template /etc/ssl/umd.info --outfile /etc/ssl/certs/umd_slapd_cert.pem
```

17) Create an LDIF file describing the server certificate info by creating a "/etc/ssl/certinfo.ldif" file:

```
vagrant> sudo vi /etc/ssl/certinfo.ldif
```

with the following contents:

```
dn: cn=config
add: olcTLSCACertificateFile
olcTLSCACertificateFile: /etc/ssl/certs/cacert.pem
-
add: olcTLSCertificateFile
olcTLSCertificateFile: /etc/ssl/certs/umd_slapd_cert.pem
-
add: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/ssl/private/umd_slapd_key.pem
```

18) Load the "/etc/ssl/certinfo.ldif" file into LDAP:

```
vagrant> sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f /etc/ssl/certinfo.ldif
```

19) Set the permissions on the files:

```
vagrant> sudo adduser openldap ssl-cert
vagrant> sudo chgrp ssl-cert /etc/ssl/private/umd_slapd_key.pem
vagrant> sudo chmod g+r /etc/ssl/private/umd_slapd_key.pem
vagrant> sudo chmod o-r /etc/ssl/private/umd_slapd_key.pem
```

20) Set the slapd configuration to run ldaps by default:

```
vagrant> sudo vi /etc/default/slapd
```

and change the line:

```
SLAPD_SERVICES="ldap:/// ldapi:///"
```

to

```
SLAPD_SERVICES="ldaps:/// ldap:/// ldapi:///"
```
 

20) Restart OpenLDAP

```
vagrant> sudo service slapd restart
```

## phpLDAPadmin installation

The "phpLDAPadmin" tool is useful for adding/modifying/deleting entries in the LDAP server. These steps are largely taken from the "Install PHPldapadmin" section in <https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-a-basic-ldap-server-on-an-ubuntu-12-04-vps>

1) Install phpLDAPadmin through apt-get:

```
vagrant> sudo apt-get install phpldapadmin
```

2) Edit the "/etc/phpldapadmin/config.php" file:

```
vagrant> sudo vi /etc/phpldapadmin/config.php
```

and make the following changes:

| Old Line                                                            | New Line                                                        |
|---------------------------------------------------------------------|-----------------------------------------------------------------|
| $servers->setValue('server','host','127.0.0.1');                    | $servers->setValue('server','host','192.168.33.10');            |
| $servers->setValue('server','base',array('dc=example,dc=com'));     | $servers->setValue('server','base',array('dc=umd,dc=edu'));     |
| $servers->setValue('login','bind_id','cn=admin,dc=example,dc=com'); | $servers->setValue('login','bind_id','cn=admin,dc=umd,dc=edu'); |

Also uncomment the following line, and make the change:

| Old Line                                                         | New Line                                                     |
|------------------------------------------------------------------|--------------------------------------------------------------|
| // $config->custom->appearance['hide_template_warning'] = false; | $config->custom->appearance['hide_template_warning'] = true; |

You can then login at <http://192.168.33.10/phpldapadmin>, with the following credentials:

| Field    | Value                  |
|----------|------------------------|
| Login DN | cn=admin,dc=umd,dc=edu |
| Password | admin                  |


## License

These files are provided under the CC0 1.0 Universal license (http://creativecommons.org/publicdomain/zero/1.0/).
