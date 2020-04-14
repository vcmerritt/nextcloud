# NextCloud Installation Instructions including NGINX
### Prerequisites

- A server running Debian 10.
- The ability to make DNS Changes


## Set the System Host Name and IP Address
``` bash
#Change the hostname of the VM before installing NextCloud
sed -i 's/BaseVMBuild/NextCloud01/g' /etc/hosts
sed -i 's/BaseVMBuild/NextCloud01/g' /etc/hostname

Change the Network to use a static IP Address (NextCloud01)
#Make sure you change the IP, mask and gateway to the correct IP before executing this command
sed -i 's/dhcp/static\n   address 192\.168\.2\.45\n   netmask 255\.255\.255\.0\n   gateway 192\.168\.2\.1\n   dns-nameservers 192\.168\.2\.40\n   dns-domain mydomain\.com\n   dns-search mydomain\.com/g' /etc/network/interfaces

#Reboot Server to make the change take effect
/usr/sbin/reboot
```

#### Install Apache, MariaDB and PHP 

NextCloud is a website that will be hosted on Apache2.  It is written in PHP, and uses MariaDB/MySQL to store data specific to the different users and server configuration. To support the installation of NextCloud, the Debian 10 system will need Apache2, MariaDB, PHP and other supporting packages on your system. <br><br>Start by installing the initial packages we will need using the following command:

``` bash
apt-get install apache2 libapache2-mod-php mariadb-server php-xml php-ldap php-cli php-cgi php-mysql php-mbstring php-gd php-curl smbclient php-zip wget unzip gnupg2 -y
```
Once all the packages are installed, modify the php.ini file to change some values to the minimum recommended settings:
``` bash

sed -i 's/memory_limit = 128/memory_limit = 512/g' /etc/php/7.3/apache2/php.ini
sed -i 's/upload_max_filesize = 2M/upload_max_filesize = 2000M/g' /etc/php/7.3/apache2/php.ini
sed -i 's/post_max_size = 8M/post_max_size = 2000M/g' /etc/php/7.3/apache2/php.ini
sed -i 's/max_execution_time = 30/max_execution_time = 300/g' /etc/php/7.3/apache2/php.ini
sed -i 's/;date\.timezone =/date\.timezone = America\/Edmonton'/g /etc/php/7.3/apache2/php.ini
```

Save and close the file when you are finished. Then, start the Apache and MariaDB service and enable them to start after system reboot with the following command:
``` bash
systemctl start apache2
systemctl start mariadb
systemctl enable apache2
systemctl enable mariadb
 ```
Once you are done, you can proceed to the next step.

### Configure Database for NextCloud
Create a database and database user for NextCloud.
``` bash
mysql -u root -p
Provide your root password when asked then create a database and user with the following commands:

CREATE DATABASE nextclouddb;
CREATE USER 'nextclouduser'@'localhost' IDENTIFIED BY 'password';

Grant all the privileges to the nextclouddb with the following command:
GRANT ALL ON nextclouddb.* TO 'nextclouduser'@'localhost';

Flush the privileges and exit from the MariaDB shell with the following command:
FLUSH PRIVILEGES;
EXIT;

Once you are done, you can proceed to the next step.
```

Download NextCloud
First, visit the NextCloud download page and download the latest version of the NextCloud on your system. At the time of writing this article, the latest version of NextCloud is 17.0.1. You can download it with the following command:
``` bash
wget https://download.nextcloud.com/server/releases/nextcloud-18.0.3.zip
```
Once the download is completed, unzip the downloaded file with the following command:

``` bash
unzip nextcloud-18.0.3.zip
```

Next, move the extracted directory to the Apache web root directory and create a data directory:
``` bash
mv nextcloud /var/www/html/
mkdir /mnt/data
chown -R www-data:www-data /mnt/data
```

Give proper permissions to the nextcloud directory with the following commands:
``` bash
chown -R www-data:www-data /var/www/html/nextcloud/
chmod -R 755 /var/www/html/nextcloud/

```

Configure Apache for NextCloud
Create an Apache virtual host configuration file to serve NextCloud. You can create it with the following command:
``` bash
cat <<EOF > /etc/apache2/sites-available/nextcloud.conf
<VirtualHost *:80>
     ServerAdmin admin@example.com
     DocumentRoot /var/www/html/nextcloud/
     ServerName nextcloud.example.com

     Alias /nextcloud "/var/www/html/nextcloud/"

     <Directory /var/www/html/nextcloud/>
        Options +FollowSymlinks
        AllowOverride All
        Require all granted
          <IfModule mod_dav.c>
            Dav off
          </IfModule>
        SetEnv HOME /var/www/html/nextcloud
        SetEnv HTTP_HOME /var/www/html/nextcloud
     </Directory>

     ErrorLog ${APACHE_LOG_DIR}/error.log
     CustomLog ${APACHE_LOG_DIR}/access.log combined

</VirtualHost>
EOF
```
Save and close the file when you are finished. Then, enable the Apache virtual host file and other required modules using the following commands:

``` bash
export PATH=$PATH:/usr/sbin
a2ensite nextcloud.conf
a2enmod rewrite
a2enmod headers
systemctl restart apache2

-- These next a2enmod statements are not necessary on Deb 10 but may be required on other versions so they are included here --
a2enmod env
a2enmod dir
a2enmod mime
Finally, restart the Apache service to apply the new configuration:

systemctl restart apache2
```

### Connect to NextCloud and Finish Configuration
In this step you will need to get the IP Address of the NextCloud system you are building to finish the configuration. To start with get the IP Address of the NextCloud system by typing ip addr:

``` bash
ip addr
```

**Open NextCloud in a web browser**
<br>
Navigate to the NextCloud server using the IP Address obtained above, and finish the configuration by supplying the details requested.  Make sure you specify /mnt/data as the data directory.   This directory can be an ISCSI mount, NFS, or another local disk / virtual disk or if you are just testing you can leave it on the same volume as NextCloud (for testing only). 

```
http://IPADDRofNextCloudServer/nextcloud
```

Pick a NextCloud Admin account user name, and password, then enter the database information using the information from the commands you executed when you used mysql to create the database (see commands above to jog your memory).

## Install Collabera Online Document Editor Solution
This will allow you to create and edit documents online from within NextCloud via a web browser (similar to google docs) using LibreOffice Online.   

**Libre Office Online (LOOLWSD) Installation**
``` bash 
wget https://www.collaboraoffice.com/repos/CollaboraOnline/CODE-centos7/repodata/repomd.xml.key && apt-key add repomd.xml.key
echo 'deb https://www.collaboraoffice.com/repos/CollaboraOnline/CODE-debian10 ./' >> /etc/apt/sources.list
apt update && apt install loolwsd code-brand -y
mkdir /opt/lool/jails
chown lool:lool /opt/lool -R
touch /var/log/loolwsd.log 
chown lool:lool /var/log/loolwsd.log
systemctl restart loolwsd
```

## Optimizations for NextCloud  - Complete after NextCloud initialization is finished.
``` bash
apt-get install sudo php-intl php-imagick php-apcu -y
sudo -u www-data php /var/www/html/nextcloud/occ db:add-missing-indices

#Answer Y when prompted during the execution of the following command:
sudo -u www-data php /var/www/html/nextcloud/occ db:convert-filecache-bigint    

#Execute the following command to enable memory caching
sed -i 's/  '"'installed'"' => true,/  '"'memcache\.local'"' => '"'\\\\OC\\\\\Memcache\\\\\APCu'"',\n  '"'installed'"' => true,/g' /var/www/html/nextcloud/config/config.php

Note:  The command above will add the following line to the config.php file for nextcloud before the line that states installed => true
   'memcache.local' => '\OC\Memcache\APCu',
   
#php.ini additional settings
sed -i 's/;opcache\.enable=1/opcache\.enable=1\napc\.enable_cli=1/g' /etc/php/7.3/apache2/php.ini
sed -i 's/;opcache\.interned_strings_buffer=8/opcache\.interned_strings_buffer=8/g' /etc/php/7.3/apache2/php.ini
sed -i 's/;opcache.max_accelerated_files=10000/opcache\.max_accelerated_files=10000/g' /etc/php/7.3/apache2/php.ini
sed -i 's/;opcache\.memory_consumption=128/opcache\.memory_consumption=128/g' /etc/php/7.3/apache2/php.ini
sed -i 's/;opcache\.save_comments=1/opcache\.save_comments=1/g' /etc/php/7.3/apache2/php.ini
sed -i 's/;opcache.revalidate_freq=2/opcache\.revalidate_freq=1/g' /etc/php/7.3/apache2/php.ini
# The sed lines above change the following opcache settings
apc.enable_cli=1
opcache.enable=1
opcache.interned_strings_buffer=8
opcache.max_accelerated_files=10000
opcache.memory_consumption=128
opcache.save_comments=1
opcache.revalidate_freq=1

#Restart apache to make the php changes take effect.
systemctl restart apache2
```

# Join the Domain and Configure Sudoers
## Install additional components
``` bash
apt-get install oddjob-mkhomedir realmd sssd-tools sssd libnss-sss libpam-sss adcli sssd-krb5 krb5-config krb5-user libpam-krb5 sudo -y
```

## Configure /etc/krb5.conf
Copy this text to a notepad document and change the occurrences of MYDOMAIN.COM or mydomain.com to the correct domain name you are creating for your environment.
``` bash
cat <<EOF > /etc/krb5.conf
[libdefaults]
        default_realm = MYDOMAIN.COM
        rdns=false

# The following krb5.conf variables are only for MIT Kerberos.
        kdc_timesync = 1
        ccache_type = 4
        forwardable = true
        proxiable = true

        fcc-mit-ticketflags = true

[realms]
        MYDOMAIN.COM = {
                kdc = SAMBADC01.mydomain.com
                kdc = SAMBADC02.mydomain.com
                admin_server = SAMBADC01.mydomain.com
                default_domain = mydomain.com
        }

[domain_realm]
        .mydomain.com = MYDOMAIN.COM
EOF
```
## Modify PAM settings to enable auto-creation of home directorys for Active Directory users
``` bash
echo "session optional      pam_oddjob_mkhomedir.so skel=/etc/skel" >> /etc/pam.d/common-session
```

## Configure DNS Resolution and Join Domain
```
#Modify resolv.conf to point to AD Domain Controller(s):
#Change /etc/resolv.conf

echo domain mydomain.com > /etc/resolv.conf
echo search mydomain.com >> /etc/resolv.conf
echo nameserver 192.168.2.40 >> /etc/resolv.conf
echo nameserver 192.168.2.41 >> /etc/resolv.conf
```

## Join the domain
``` bash
/usr/sbin/realm join --user=administrator mydomain.com --install=/
```
``` bash
## Modify Configuration Files
#Tweak the /etc/sssd/sssd.conf file to enable authentication to the newly installed AD
cat <<EOF > /etc/sssd/sssd.conf 
[sssd]
domains = MYDOMAIN.COM
config_file_version = 2
services = nss, pam

[domain/MYDOMAIN.COM]
ad_domain = mydomain.com
krb5_realm = MYDOMAIN.COM
realmd_tags = manages-system joined-with-adcli
cache_credentials = True
ad_maximum_machine_account_password_age = 30
id_provider = ad
krb5_store_password_if_offline = True
default_shell = /bin/bash
ldap_id_mapping = True
ldap_schema = ad
auto_private_groups = true
dyndns_update = true
dyndns_refresh_interval = 43200
dyndns_update_ptr = true
dyndns_ttl = 3600
use_fully_qualified_names = False
fallback_homedir = /home/%u@%d
access_provider = ad
id_provider = ad
auth_provider = ad
chpass_provider = ad
EOF
```

## Configure /etc/samba/smb.conf for client connections
``` bash 
cat <<EOF > /etc/samba/smb.conf
 [global]
    netbios name = NEXTCLOUD01
    security = ADS
    workgroup = TESTDOMAIN
    kerberos method = secrets and keytab
    realm = TESTDOMAIN.COM

    log file = /var/log/samba/%m.log
    log level = 1

    # Default idmap config used for BUILTIN and local windows accounts/groups
    idmap config *:backend = tdb
    idmap config *:range = 2000-9999

    # idmap config for domain TESTDOM
    idmap config TESTDOMAIN:backend = rid
    idmap config TESTDOMAIN:range = 10000-99999

    # Use template settings for login shell and home directory
    winbind nss info = template
    template shell = /bin/bash
    template homedir = /home/%U

    winbind enum users = Yes
    winbind enum groups = Yes
    winbind use default domain = Yes
    encrypt passwords = Yes
EOF
```

## Configure Sudoers to authorize users based on AD Groups
``` bash
#Add Active Directory Group to sudoers file
echo '%linuxsudoers           ALL=(ALL)       ALL' >> /etc/sudoers

#The goal is to use the group above, but you can also add the Administrators group to allow any user on the Domain Controller that is in that group to use sudo.
echo '%domain\ admins	         ALL=(ALL)	       ALL' >> /etc/sudoers

#Restart services 
systemctl restart sssd

#Use kinit to connect as administrator and update the /etc/krb5.keytab
kinit administrator
net ads keytab create

#Trigger DNS Dynamic Updates
net ads dns register -P

Note:  If the DNS Dynamic updates fails, then validate the system hostname in /etc/hosts and ensure the fully qualified name is correct.

#MAKE SURE TO LOG OUT OF ALL Putty Sessions after making the changes to the SUDOERS group and then use putty to log back into the DC and test things out. 
```

##  Fix Access.php to support LDAP Groups 
The Access.php file is broken in NextCloud 18.02, and 18.03 but can be fixed by replacing Access.php as per the following instructions.
``` bash
# Login to the server via Putty to fix the LDAP Groups issue as there is a bug in the Access.php shipped with NextCloud 18.02
cd /var/www/html/nextcloud/apps/user_ldap/lib
mv Access.php Access.php.bak
wget https://raw.githubusercontent.com/nextcloud/server/5bf3d1bb384da56adbf205752be8f840aac3b0c5/apps/user_ldap/lib/Access.php
```

##  Copy Samba Domain CA Certificates to support LDAPS
Use Putty to connect to the NextCloud server.

``` bash
#Change to the root user home directory
cd ~/

#Connect to the Domain Controllers as the domain administrator and copy the DC Certificates locally to the NextCloud Server
#YOU WILL NEED TO ANSWER ALL PROMPTS INCLUDING SPECIFYING THE DOMAIN ADMIN PASSWORD TWICE
scp administrator@192.168.2.40:/var/lib/samba/private/tls/ca.pem ./DC1CA.crt
scp administrator@192.168.2.41:/var/lib/samba/private/tls/ca.pem ./DC2CA.crt
sudo cp *.crt /usr/local/share/ca-certificates
sudo /usr/sbin/update-ca-certificates

#Copy the certificates to the NextCloud Cert file in /var/www/html/nextcloud/resources/config/ca-bundle.crt
#First create the entry for DC01
echo "" >> /var/www/html/nextcloud/resources/config/ca-bundle.crt
echo "DC01 Active Directory CA Cert" >> /var/www/html/nextcloud/resources/config/ca-bundle.crt
echo "=========================================" >> /var/www/html/nextcloud/resources/config/ca-bundle.crt
cat DC1CA.crt >> /var/www/html/nextcloud/resources/config/ca-bundle.crt

#Now create the entry for DC02
echo "" >> /var/www/html/nextcloud/resources/config/ca-bundle.crt
echo "DC02 Active Directory CA Cert" >> /var/www/html/nextcloud/resources/config/ca-bundle.crt
echo "=========================================" >> /var/www/html/nextcloud/resources/config/ca-bundle.crt
cat DC2CA.crt >> /var/www/html/nextcloud/resources/config/ca-bundle.crt

#Reboot the NextCloud Server 
/usr/sbin/reboot

```
## Install NGINX to enable SSL
Navigate to the following github url and follow the instructions to install NGINX on the NextCloud server.

https://github.com/vcmerritt/nginx
