# nextcloud
NextCloud Installation Instructions including NGINX
Prerequisites
A server running Debian 10.
A valid domain name pointed with your server IP. in this tutorial, we will use nextcloud.example.com domain.
A root password is configured on your server.
Install Apache, MariaDB and PHP
NextCloud runs on the webserver, written in PHP and uses MariaDB to store their data. So you will need to install Apache, MariaDB, PHP and other required packages on your system. You can install all of them by running the following command:

apt-get install apache2 libapache2-mod-php mariadb-server php-xml php-cli php-cgi php-mysql php-mbstring php-gd php-curl php-zip wget unzip -y
Once all the packages are installed, open the php.ini file and tweak some recommended settings:

nano /etc/php/7.3/apache2/php.ini
Change the following settings:

memory_limit = 512M
upload_max_filesize = 500M
post_max_size = 500M
max_execution_time = 300
date.timezone = Asia/Kolkata
Save and close the file when you are finished. Then, start the Apache and MariaDB service and enable them to start after system reboot with the following command:

systemctl start apache2
 systemctl start mariadb
 systemctl enable apache2
 systemctl enable mariadb
Once you are done, you can proceed to the next step.

Configure Database for NextCloud
Next, you will need to create a database and database user for NextCloud. To do so, log in to MariaDB shell with the following command:

mysql -u root -p
Provide your root password when asked then create a database and user with the following command:

MariaDB [(none)]> CREATE DATABASE nextclouddb;
 MariaDB [(none)]> CREATE USER 'nextclouduser'@'localhost' IDENTIFIED BY 'password';
Next, grant all the privileges to the nextclouddb with the following command:
MariaDB [(none)]> GRANT ALL ON nextclouddb.* TO 'nextclouduser'@'localhost';
Next, flush the privileges and exit from the MariaDB shell with the following command:

MariaDB [(none)]> FLUSH PRIVILEGES;
 MariaDB [(none)]> EXIT;
Once you are done, you can proceed to the next step.

Download NextCloud
First, visit the NextCloud download page and download the latest version of the NextCloud on your system. At the time of writing this article, the latest version of NextCloud is 17.0.1. You can download it with the following command:

wget https://download.nextcloud.com/server/releases/nextcloud-17.0.1.zip
Once the download is completed, unzip the downloaded file with the following command:

unzip nextcloud-17.0.1.zip
Next, move the extracted directory to the Apache web root directory:
mv nextcloud /var/www/html/
Next, give proper permissions to the nextcloud directory with the following command:

chown -R www-data:www-data /var/www/html/nextcloud/
 chmod -R 755 /var/www/html/nextcloud/
Once you are finished, you can proceed to the next step.

Configure Apache for NextCloud
Next, you will need to create an Apache virtual host configuration file to serve NextCloud. You can create it with the following command:

nano /etc/apache2/sites-available/nextcloud.conf
Add the following lines:

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
Save and close the file when you are finished. Then, enable the Apache virtual host file and other required modules using the following commands:
Advertisements


a2ensite nextcloud.conf
 a2enmod rewrite
 a2enmod headers
 a2enmod env
 a2enmod dir
 a2enmod mime
Finally, restart the Apache service to apply the new configuration:

systemctl restart apache2
