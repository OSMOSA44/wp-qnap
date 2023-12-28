# Wordpress on QNAP integration

This project is intended to explain how to configure multiple Wordpress sites on QNAP NAS server using

* Container Station service to run Wordpress container and his associated MariaDB database container
* Virtual Host Web Server manager to access Wordpress sites
* SSL Certificate & Private Key service to manage TLS Certificate (Adding Alternative Name)

NOTA: Integration realized on QNAP NAS TS-253A with version QTS 5.1.4.2596 (2023/11/28)

## DNS Setting

Declare Type A entry with public QNAP NAS IP address (static IP address of your Internet Box in general) for each Virtual Host URL

Example to redirect `rec.mydomain.com` and `www.mydomain.com` to QNAP NAS (with Internet Box Ip address 1.2.3.4) :

```text
rec 10800 IN A 1.2.3.4
www 10800 IN A 1.2.3.4
```

This step is needed before adding Alternative Names on Let's Encrypt Certificate demand

CN: <nas_name>.myqnapcloud.com
eMail: <nas_mail_admin>
SAN: rec.mydomain.com,www.mydomain.com

NOTA: You can add SAN's with differents domain names

## NAS QNAP perequesites

To avoid permission errors on mapped volumes, we create mysql(999) and www-data(33) users, directories on QNAP Host and change directories owner

```bash
# You have to access your NAS via SSH connection (terminal window)
ssh -o ServerAliveInterval=120 -o ServerAliveCountMax=2 -p 4522 admin@<nas_name>.myqnapcloud.com
cat /etc/group # check existing group
cat /etc/passwd # check mysql(999) and www-data(33) does not already exist

# Baseline Directory and user for Wordpress Containers
userdel www-data # if needed (be caution !)
mkdir -p /share/Container/www-data/rec.mydomain.com /share/Container/www-data/www.mydomain.com
addgroup -g 33 www-data
useradd -u 33 -g 33 -d /share/Container/www-data -M -c "DO_NOT_ERASE_ME" www-data
chown -R www-data:www-data /share/Container/www-data

# Baseline Directory and user for MariaDB Containers
userdel mysql # if needed (be caution !)
mkdir -p /share/Container/mysql/rec.mydomain.com /share/Container/mysql/www.mydomain.com
addgroup -g 999 mysql
useradd -u 999 -g 999 -d /share/Container/mysql -M -c "DO_NOT_ERASE_ME" mysql
chown -R mysql:mysql /share/Container/mysql

# Check New users creation
cat /etc/passwd
www-data:x:33:33:DO_NOT_ERASE_ME:/share/Container/www-data:
mysql:x:999:999:DO_NOT_ERASE_ME:/share/Container/mysql:
```

## MariaDB (mysql user)

* Create Wordpress database at image startup through MARIADB_DATABASE environment variable
* Create user for the MARIADB_DATABASE database by setting MARIADB_USER and MARIADB_PASSWORD environment variables

## Wordpress (www-data user)

* We use php-fpm image version, without httpd apache server, we will use the integrated QNAP NAS httpd apache server to expose Wordpress sites
* The WORDPRESS_DB_NAME (== MARIADB_DATABASE) needs to already exist on the given MySQL server; it will not be created by the wordpress container

## Nginx

* We use unpriviledged docker image `nginxinc/nginx-unprivileged:1.25.3`
* We map nginx.conf file
* We map Wordpress files directory

### Set Host NAS to prepare volume mapping

```bash
mkdir -p /share/Container/nginx/rec.mydomain.com /share/Container/nginx/www.mydomain.com
cd /share/Container/nginx/rec.mydomain.com
vi nginx.conf # insert nginx/nginx-rec.mydomain.com.conf content file
chmod 644 nginx.conf
cd /share/Container/nginx/www.mydomain.com
vi nginx.conf # insert nginx/nginx-www.mydomain.com.conf content file
chmod 644 nginx.conf
chown -R www-data:www-data /share/Container/nginx
```

## Apache Virtual Host

* Create virtualhost configuration file for each web site
* Modify `apache.conf` file to add new virtualhost
* Restart QNAP NAS Web service (Apache Web Server)

### Virtualhost Config Files

* Create new files and paste `apache/rec.mydomain.com.conf` and `apache/www.mydomain.com.conf` contents respectively

```bash
[~] # cd /etc/config/apache/
[/etc/config/apache] # ll
total 172K
drwxr-xr-x  4 admin administrators 4.0K 2023-12-01 03:30 ./
drwxr-xr-x 56 admin administrators  12K 2023-12-01 11:23 ../
-rw-r--r--  1 admin administrators 6.2K 2023-11-30 20:38 apache.conf
drwxr-xr-x  2 admin administrators 4.0K 2023-12-01 02:38 extra/
-rw-r--r--  1 admin administrators  13K 2016-08-31 20:53 magic
-rw-r--r--  1 admin administrators  60K 2023-11-27 17:58 mime.types
drwxr-xr-x  3 admin administrators 4.0K 2016-08-31 20:53 original/
-rw-r--r--  1 admin administrators  445 2019-01-24 23:19 php-fpm.conf
-rw-r--r--  1 admin administrators  445 2023-08-04 00:26 php-fpm-qweb.conf
-rw-r--r--  1 admin administrators  687 2023-12-01 03:30 rec.mydomain.com.conf
-rw-r--r--  1 admin administrators  687 2023-12-01 03:30 www.mydomain.com.conf
```

### Modify Apache Web Server Configuration

Insert new virtualhost web server files to the end of `apache.conf` file but before others 'Include' lines

```bash
...
<IfModule reqtimeout_module>
        RequestReadTimeout header=20-40,MinRate=500 body=20,MinRate=500
</IfModule>
Include /etc/config/apache/www.mydomain.com.conf
Include /etc/config/apache/rec.mydomain.com.conf
Include /etc/config/apache/extra/apache-ssl.conf
Include /etc/config/apache/extra/apache-fastcgi.conf
Include /etc/config/apache/extra/apache-musicstation.conf
Include /etc/config/apache/extra/apache-photo.conf
Include /etc/config/apache/extra/apache-video.conf
Include /etc/config/apache/extra/apache-common.conf
Include /etc/config/apache/extra/httpd-vhosts-user.conf
Include /etc/config/apache/extra/httpd-ssl-vhosts-user.conf
Include /etc/config/apache/extra/apache-qmail.conf
```

### Restart QNAP NAS WEB Service

```bash
/etc/init.d/Qthttpd.sh
echo "Usage: /etc/init.d/$0 {start|stop|restart|reload_apache|qstop|apache_stop|apache_start}"
/etc/init.d/Qthttpd.sh reload_apache
```

* Apache Binary Location

```bash
[~] # cd /usr/local/apache
[/usr/local/apache] # ll
total 40K
drwxr-xr-x 10 admin administrators 4.0K 2023-11-27 17:47 ./
drwxr-xr-x 33 admin administrators 4.0K 2023-11-28 14:14 ../
drwxr-xr-x  2 admin administrators 4.0K 2023-11-28 14:14 bin/
drwxr-xr-x  2 admin administrators 4.0K 2023-11-09 21:44 cgi-bin/
lrwxrwxrwx  1 admin administrators   18 2023-11-27 17:47 conf -> /etc/config/apache/
drwxr-xr-x  3 admin administrators 4.0K 2023-11-09 21:58 lib/
drwxr-xr-x  2 admin administrators 4.0K 2023-11-09 21:58 links/
drwxr-xr-x  5 admin administrators 4.0K 2023-11-30 16:43 logs/
drwxr-xr-x  6 admin administrators 4.0K 2023-11-30 19:11 md/
drwxr-xr-x  2 admin administrators 4.0K 2023-11-09 21:58 modules/
drwxr-xr-x  3 admin administrators 4.0K 2023-11-09 21:44 share/
```

## QNAP NAS Application

* Connect to QNAP NAS Administration interface
* Open 'Container Station' service
* Select 'Applications' and click on +Create button
* Set Application name as the sevice field in docker-compose file
* Copy and Paste the docker-compose content file
* Click on Create button

### docker-compose.yml location

```bash
/share/Container/container-station-data/application/
```
