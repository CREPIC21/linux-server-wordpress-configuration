# Linux Server Configuration - Setting Up a Wordpress Site
 - VM, SSH, DNS, VPS, WEB, PHP, MySql, WordPress

> :mega: In the configuration steps replace all occurrences of:
> - `wordpresslinux.xyz` with your chosen domain name
> - `wordpress` with your own chosen word
> - `vm_ip_address` with your VM IP address


## **1. Create VM in Azure or any other cloud provider(Digital Ocean, Linode, AWS...)**
- use Ubuntu Server 20.04 image
- use password for authentication
- once VM is created, connect to it using SSH: 
```
ssh -p 22 <user_name>@<vm_ip_address>
```
- as root run: 
```
apt update && apt full-upgrade -y
```

## **2. Secure SSH with Key Authentication**
- on your local machine(laptop/PC) that you will use to SSH to the VM create key pairs in `/home/wordpress/` directory:
```
ssh-keygen -t rsa -b 2048 -C 'linux server VM keys'
```
- add created public key to your VM machine `home` directory by copy/pasting it to the VM `.ssh/authorized_keys` file:
```
vim .ssh/authorized_keys
```
- disable password authentication on the server(`PasswordAuthentication no`) and restart the SSH service:
```
vim /etc/ssh/sshd_config

// set PasswordAuthentication to no in the configuration file
PasswordAuthentication no

systemctl restart ssh
```
- now you can SSH to the VM without password(you will not be asked for a password anymore): 
```
ssh -p 22 <user_name>@<vm_ip_address>
```


## **3. Getting a Domain name**
- for this example we used [Namecheap](https://ap.www.namecheap.com/) and domain name `wordpresslinux.xyz`
- navigate to: `Domain List -> manage (next to domain wordpresslinux.xyz) -> Advanced DNS` and under `PERSONAL DNS SERVER` add two nameservers:
```
ns1 with VM IP address
ns2 with VM IP address
```
- navigate to the `DOMAIN` tab and under `NAMESERVERS` choose Custom DNS and add created nameservers:
```
ns1.wordpresslinux.xyz
ns2.wordpresslinux.xyz
```
- in can take up to 72 hours for change to take an effect, to check run in terminal:
```
dig -t ns wordpresslinux.xyz
```

## **4. Installing a DNS Server (Bind9)**
- SSH to VM:
```
ssh -p 22 <user_name>@<vm_ip_address>
```
- as root run: 
```
apt update && apt install bind9 bind9utils bind9-doc
```
- check if the service is running: 
```
systemctl status bind9
```
- set IPv4 since we are using only IPv4 mode, add `-4` to the end of options parameter in `/etc/default/named` file and restart the service:
```
vim /etc/default/named

// add `-4` to the end of options parameter in the configuration
OPTIONS="-u bind -4"

systemctl restart bind9
```
- testing:
  - sending DNS query to DNS server that is now running on a localhost: `dig -t a @localhost google.com`
  - sending DNS query to DNS server that is public from CloudFlare: `dig -t a @1.1.1.1 google.com`
- add two forwarders to our DNS Server in the `/etc/bind/named.conf.options` file, set public google DNS servers as forwarders for my server and restart the service:
```
vim /etc/bind/named.conf.options

// configuration to add
forwarders {
      8.8.8.8;
      8.8.4.4;
};

systemctl restart bind9
```
- if the server doesn't know the DNS information asked by the clients, it will send a recursive query to those two forwarder servers and wait for a final answer and if it gets the answer, it will pass it to the client that has asked for it in the first place
- testing: 
```
dig -t a @localhost kali.org
```

## **5. Setting up the Authoritative BIND9 DNS Server**
- open `/etc/bind/named.conf.local` and add the following configuration:
```
vim /etc/bind/named.conf.local

// configuration to add
zone "wordpresslinux.xyz" {
        type master;
        file "/etc/bind/db.wordpresslinux.xyz";
};
```
- in the above configuration we created a new zone using the zone clause and specifyed that this is the master zone, which means that this is the master of authoritative DNS server for the Domain, the file is db.wordpresslinux.xyz, where we'll add DNS records for the domain
- next step is to create the zone file and instead of creating the zone file from scratch, we will use a zone template file which exists in `/etc/bind`:
```
cp /etc/bind/db.empty /etc/bind/db.wordpresslinux.xyz
vim /etc/bind/db.wordpresslinux.xyz

// configuration to add
; BIND reverse data file for empty rfc1918 zone
;
; DO NOT EDIT THIS FILE - it is used for multiple zones.
; Instead, copy it, edit named.conf, and use that copy.
;
$TTL    86400
@       IN      SOA     ns1.wordpresslinux.xyz. root.localhost. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                          86400 )       ; Negative Cache TTL
;
@       IN      NS      ns1.wordpresslinux.xyz.
ns1     IN      A       vm_ip_address
mail    IN      MX 10   mail.wordpresslinux.xyz.
wordpresslinux.xyz.     IN      A vm_ip_address
www     IN      A       vm_ip_address
mail    IN      A       vm_ip_address
```
- before restarting the server, check if there are any syntax errors in the configuration files:
```
named-checkconf
named-checkzone wordpresslinux.xyz /etc/bind/db.wordpresslinux.xyz
``` 
- silent output indicates no errors were found, if there are syntax errors in the zone file, you need to fix them or the zone won't be loaded
- restart the service:
```
systemctl restart bind9
```
- check the service status and if the zone was loaded successfully:
```
systemctl status bind9
```
- always update the SOA serial number when you make changes to a zone file:
```
vim /etc/bind/db.wordpresslinux.xyz

// on line 8 we changed SOA serial number from  1 to 2
; BIND reverse data file for empty rfc1918 zone
;
; DO NOT EDIT THIS FILE - it is used for multiple zones.
; Instead, copy it, edit named.conf, and use that copy.
;
$TTL    86400
@       IN      SOA     ns1.wordpresslinux.xyz. root.localhost. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                          86400 )       ; Negative Cache TTL
;
@       IN      NS      ns1.wordpresslinux.xyz.
ns1     IN      A       vm_ip_address
mail    IN      MX 10   mail.wordpresslinux.xyz.
wordpresslinux.xyz.     IN      A vm_ip_address
www     IN      A       vm_ip_address
mail    IN      A       vm_ip_address
```
- testing: 
```
dig @localhost -t ns wordpresslinux.xyz
dig @localhost -t a www.wordpresslinux.xyz
```
- what we've done so far was setting up the primary master and the direct resolution, which means translating domains to IP addresses, and the reverse resolution, which is optional, means translating IP addresses to domain names

## **6. Installing a Web Server(Apache2)**
```
apt update && apt install apache2
systemctl status apache2
systemctl enable apache2
```
- if ufw is active run:
```
ufw status
ufw allow 'Apache Full'
```
- testing
  - open web browser and go to the VM IP address, trick to get IP address from terminal: `curl -4 ident.me`
  - open web browser and go to the VM domain name `wordpresslinux.xyz`

## **6.1 SetHandler and Server Status for Apache2**
- enable `mod_status`:
```
a2enmod status
```
- next step is to configure it and add my public IP address to the white listed IP addresses in `Location` directive:
```
vim /etc/apache2/mods-avaliable/status.conf
```
```
<IfModule mod_status.c>
        # Allow server status reports generated by mod_status,
        # with the URL of http://servername/server-status
        # Uncomment and change the "192.0.2.0/24" to allow access from other hosts.

        <Location /server-status>
                SetHandler server-status
                Require local
                #Require ip 192.0.2.0/24
                Require ip vm_ip_address
        </Location>

        # Keep track of extended status information for each request
        ExtendedStatus On

        # Determine if mod_status displays the first 63 characters of a request or
        # the last 63, assuming the request itself is greater than 63 chars.
        # Default: Off
        #SeeRequestTail On


        <IfModule mod_proxy.c>
                # Show Proxy LoadBalancer status in mod_status
                ProxyStatus On
        </IfModule>
</IfModule>
# vim: syntax=apache ts=4 sw=4 sts=4 sr noet
```
- reload the apache server: 
```
systemctl restart apache2
```
- in the browser: `https://www.wordpresslinux.xyz/server-status` is displaying useful information about the server and its connections
- to get the status page to update itself automatically add `?refresh=n`, where n means refresh every n seconds
  - example: `https://www.wordpresslinux.xyz/server-status?refresh=2` where the page will be refreshed automatically every two seconds

## **7. Setting up Virtual Hosting for multiple websites(domain names)**
- create directory for each website in `/var/www/`:
```
mkdir /var/www/wordpresslinux.xyz
```
- check which user is running apache2:
```
ps -ef | grep apache2
```
- change the ownership of created directory:
```
chown -R www-data.www-data /var/www/wordpresslinux.xyz/
```
- change the permissions for created directory:
```
chmod 755 /var/www/wordpresslinux.xyz/
```
- create html content in created directory:
```
vim /var/www/wordpresslinux.xyz/index.html

// add some text/html such as:
Hello Wordpress!!!!
```
- navigate to apache2 configuration directory
```
cd /etc/apache2/
```
- create a virtual host file and add the configuration for created site
```
vim sites-available/wordpresslinux.xyz.conf

// configuration to add
<VirtualHost *:80>
        ServerName wordpresslinux.xyz
        ServerAlias www.wordpresslinux.xyz
        DocumentRoot /var/www/wordpresslinux.xyz

        ServerAdmin webmaster@wordpresslinux.xyz
        ErrorLog /var/log/apache2/wordpresslinux_xyz_error.log
        CustomLog /var/log/apache2/wordpresslinux_xyz_access.log combined
</VirtualHost>
```
- enable the site
```
a2ensite wordpresslinux.xyz
```
- reload the apache server
```
systemctl reload apache2
```

## **8. Securing Apache with OpenSSH and Digital Certificates**
```
apt update && apt install certbot python3-certbot-apache
certbot -d wordpresslinux.xyz
certbot -d www.wordpresslinux.xyz
systemctl status certbot.timer
certbot renew --dry-run
```

## **9. Installing PHP**
- in addition to the PHP package we we are also installing php-mysql which is a PHP module that allows PHP to communicate with MySQL
- we are also installing libapache2-mod-php which is required to enable Apache to handle PHP files
```
apt update && apt install php php-mysql libapache2-mod-php
```
- restart the service: 
```
systemctl restart apache2
```
- let's test Apache and PHP work together and are configured properly to handle dynamic content:
```
vim /var/www/wordpresslinux.xyz/test.php

// testing configuration to add
<?php
        phpinfo();
?>
```
- test it by going to browser: `ip_address/test.php`(http://vm_ip_address/test.php)

## **10. Installing and securing MySQL Server**
```
apt update && apt install mysql-server
```
- once the installation is complete, the MySQL server will start automatically:
```
systemctl status mysql
```
- the deamon process that’s running is called `mysqld`, you can check it by running: 
```
ps -ef | grep mysql
```
- MySQL is not very secure so it’s recommended to run a security script that comes pre-installed with MySQL, the script will remove some insecure default settings and lock down access to the database server by removing some MySQL accounts and setting the admin password, run command:
```
mysql_secure_installation
```
- in case of error when providing the password see this article: https://devanswe.rs/how-to-fix-failed-error-set-password-has-no-significance-for-user-rootlocalhost/?utm_content=cmp-true
- once completed login to mysql: 
```
mysql -u root -p
```

## **11. Installing and securing phpAdmin**
- before installing phpmyadmin we need to make a change to the MySQL server, if we enabled the Validate Password plugin for MySQL, it will trigger an error when you attempt to set the password for the phpmyadmin user, to solve this issue, we have to stop the Validate Password plugin from being loaded by the database server:
```
mysql -u root -p
```
```
UNINSTALL COMPONENT "file://component_validate_password";
exit
```
- install phpmyadmin along with other php extensions that are recommended in official documentation so that we can enable certain functionalities and improve performance:
```
apt update && apt install phpmyadmin php-mbstring php-zip php-gd php-json php-curl
```
- reload the apache server: 
```
systemctl restart apache2
```
- now in the browser you can login to the `phpMyAdmin`:
  - URL: `https://www.wordpresslinux.xyz/phpmyadmin`
  - username: `phpmyadmin`
  - password: `password created during the installation`
- create a dedicated MySQL user with full administrative privileges for phpmyadmin:
```
mysql -u root -p
```
```
CREATE USER 'newuser'@'localhost' IDENTIFIED WITH caching_sha2_password BY 'newuser123#@!';
GRANT ALL PRIVILEGES ON *.* TO 'newuser'@'localhost' WITH GRANT OPTION;
```
- now in the browser you can login to the `phpMyAdmin` with new created user
- to make `phpAdmin` more secure, change the URL used to access `phpMyAdmin` by changing the `Alias` directive in `/etc/phpmyadmin/apache.conf` file

## **12. Installing a Web Application(WordPress)**
- step 1 is to create a MySQL database, to store all the data like posts, pages, users, plugins or settings:
```
mysql -u root -p
```
```
CREATE DATABASE wordpress;
CREATE USER 'wordpressuser'@'localhost' IDENTIFIED BY 'Password12345.';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wordpressuser'@'localhost';
FLUSH PRIVILEGES;
exit
```
- step 2 is to download the latest version of WordPress from the official website, extract it, move the context to the domain root directory, changing the ownership...:
```
cd /tmp/
wget https://wordpress.org/latest.tar.gz
tar -xzvf latest.tar.gz
rm -rf /var/www/wordpresslinux.xyz/*
mv /tmp/wordpress/* /var/www/wordpresslinux.xyz/
ps -ef | grep apache2
chown -R www-data.www-data /var/www/wordpresslinux.xyz/
ls -l /var/www/wordpresslinux.xyz/
```
- finish the setup in the UI, go to the VM IP address:
```
http://vm_ip_address/wp-admin/
username: xxx
password: xxx
```



































































































