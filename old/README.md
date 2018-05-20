# Basic installation process of LEMP

**Last update**: 20/6/2017, tested on Ubuntu 16.04

1. [Basic installation process of LEMP](#basic-installation-process-of-lemp)
	1. [Overview](#overview)
	1. [Essentials - user, apt, default apps](#essentials)
		1. [Installation script](#installation-script)
		1. [Manual installation](#manual-script)
	1. [Webserver - PHP7.1, MariaDB, Nginx, ...](#webserver-installation)
        1. [Install Nginx](#install-nginx)
        1. [Install Apache instead of Nginx](#install-apache-instead-of-nginx-if-you-familier-with-apache)
        1. [Install MariaDB](#install-mariadb)
        1. [Install MySQL instead of MariaDB](#install-mysql-instead-of-mariadb-if-you-familier-with-mysql)
        1. [Install PHP 7.1](#install-php71)
        1. [Install PHP 7.1 Modules](#choose-and-install-php71-modules)
        1. [Install Composer](#install-composer)
	1. [Adding website](#add-new-website-configuring-php-&-nginx-&-mariadb)
		1. [User, permissions, structure...](#create-the-dir-structure-for-new-website)		1. [PHP](#create-new-php-fpm-pool-for-new-site)
		1. [Nginx vhost](#create-new-vhost-for-nginx)
        1. [Apache vhost](#create-new-vhost-for-apache)
		1. [MariaDB (MySQL)](#mariadb-mysql)
    1. [Adding Redis and MongoDB](#adding-redis-and-mongodb)
        1. [Redis](#redis)
        1. [MongoDB](#mongodb)
	1. [Todo](#todo)
	1. [Reference](#reference)
	1. [License](#license)

## Overview

This document is a list of notes when installing several Ubuntu LEMP instances w/ PHP7.1. With some sort of imagination it can be considered as a step-by-step tutorial of really basic installation process of LEMP. I wrote it mainly for myself, but feel free to use it. The LEMP consists of:

- Nginx
- Apache
- PHP7.1
- MariaDB
- MySQL

## Essentials

#### Fix locale if you are getting "WARNING! Your environment specifies an invalid locale."
```sh
sudo echo 'LC_ALL="en_US.UTF-8"' >> /etc/environment
# Log out & in
```

### Sett the correct timezone
```sh
sudo dpkg-reconfigure tzdata
```

### Configure & Update APT
```sh
sudo apt-get update ; sudo apt-get upgrade
sudo apt-get install python-software-properties
sudo apt-get install software-properties-common
```

#### Install essentials
```sh
sudo apt-get install mc
sudo apt-get install htop
```

#### Setup and configure Firewall

Open SSH port only.

```sh
sudo ufw allow OpenSSH
sudo ufw allow http
sudo ufw allow https
yes | sudo ufw enable
sudo ufw app list
```


## Webserver installation

### Install Nginx
[Installation guide from official page](https://www.nginx.com/resources/wiki/start/topics/tutorials/install/)
Create source file
```sh
sudo vim /etc/apt/sources.list.d/nginx.list
```
Put content
```sh
deb http://nginx.org/packages/ubuntu/ xenial nginx
deb-src http://nginx.org/packages/ubuntu/ xenial nginx
```

```sh
sudo apt-get update
sudo apt-get install nginx
```

If If a W: `GPG error: http://nginx.org/packages/ubuntu xenial Release: The following signatures couldn't be verified because the public key is not available: NO_PUBKEY $key is encountered` during the NGINX repository update, execute the following:
```sh
## Replace $key with the corresponding $key from your GPG error.
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys $key
sudo apt-get update
sudo apt-get install nginx
```

Check nginx service
```sh
sudo systemctl status nginx
```

If you are upgrading Ubuntu or Nginx you may have problem with masked.
Please do the following command to remove old masked
`sudo systemctl unmask nginx.service`
After that you can start Nginx as usual
### Install Apache instead of Nginx (If you familier with Apache)
```sh
sudo apt-get update
sudo apt-get install apache2
sudo systemctl status apache2
```
### Install MariaDB
```sh
sudo apt-get install mariadb-server # Or MySQL: sudo apt-get install mysql-server
sudo service mysql stop # Stop the MySQL if is running.
sudo mysql_install_db
sudo service mysql start
sudo mysql_secure_installation
```
### Install MySQL instead of MariaDB (If you familier with MySQL)
```sh
sudo apt-get update
sudo apt-get install mysql-server
sudo mysql_secure_installation
```

### Install PHP7.1
```sh
sudo add-apt-repository ppa:ondrej/php
sudo apt-get update
sudo apt-get install php7.1
```

### Choose and install PHP7.1 modules
```sh
sudo apt-cache search php7.1-*
sudo apt-get install php7.1-fpm php7.1-mysql php7.1-curl php7.1-gd php7.1-mcrypt php7.1-sqlite3 php7.1-bz2 php7.1-mbstrin php7.1-soap php7.1-xml php7.1-zip php-imagick 
```

### Speedup PHP with cache
```sh
apt-get -y install php7.1-opcache php-apcu
```

### Check the installed PHP version
```sh
php -v
```

### Restart Nginx
```sh
sudo service nginx restart ; sudo systemctl status nginx.service
```

### Or Restart Apache2
```sh
sudo service apache2 restart ; sudo systemctl status apache2
```

## Add new website, configuring PHP & Nginx & MariaDB

Steps 1. - 9. can be skipped by calling the `add-vhost.sh`. Just download `add-vhost.sh`, `chmod a+x ./add-vhost.sh` and call it `sudo ./add-vhost.sh`.
The file is deleted automatically.

### 1. Create the dir structure for new website
```sh
sudo mkdir -p /var/www/vhosts/new-website.com/{web,logs,ssl}
```

### 2. User groups and roles
```sh
sudo groupadd new-website
sudo useradd -g new-website -d /var/www/vhosts/new-website.com new-website
sudo passwd new-website
```

You can switch users by using `sudo su - new-website`


### 3. Update permissions
```sh
sudo chown -R new-website:new-website /var/www/vhosts/new-website.com
sudo chmod -R 0775 /var/www/vhosts/new-website.com
```

### 4. Create new PHP-FPM pool for new site
```sh
sudo nano /etc/php/7.0/fpm/pool.d/new-website.com.conf
```

#### 5. Configure the new pool
```sh
[new-website]
user = new-website
group = new-website
listen = /run/php/php7.1-fpm-new-website.sock
listen.owner = www-data
listen.group = www-data
php_admin_value[disable_functions] = exec,passthru,shell_exec,system
php_admin_flag[allow_url_fopen] = off
pm = dynamic
pm.max_children = 5 # The hard-limit total number of processes allowed
pm.start_servers = 2 # When nginx starts, have this many processes waiting for requests
pm.min_spare_servers = 1 # Number spare processes nginx will create
pm.max_spare_servers = 3 # Number spare processes attempted to create
pm.max_requests = 500
chdir = /
```

##### 5.1 Configuring `pm.max_children`
1. Find how much RAM FPM consumes: `ps -A -o pid,rss,command | grep php-fpm` -> second row in bytes
    1. Reference: [https://overloaded.io/finding-process-memory-usage-linux](https://overloaded.io/finding-process-memory-usage-linux)
2. Eg. ~43904 / 1024 -> ~43MB per one process
3. Calculation: If server has 2GB RAM, let's say PHP can consume 1GB (with some buffer, otherwise we can use 1.5GB): 1024MB / 43MB -> ~30MB -> pm.max_childern = 30

##### 5.2 Configuring `pm.start_servers`, `pm.min_spare_servers`, `pm.max_spare_servers`
1. `pm.start_servers` == number of CPUs
1. `pm.min_spare_servers` = `pm.start_servers` / 2
1. `pm.max_spare_servers` = `pm.start_servers` * 3

##### 5.3 Configure `/etc/nginx/nginx.conf`
```
worker_processes auto;
events {
        use epoll;
        worker_connections 1024; # ~ RAM / 2
        multi_accept on;
}
```

#### 6. Restart PHP fpm and check it's running
```sh
sudo service php7.1-fpm restart
ps aux | grep new-site
```

### 7. Create new "vhost" for Nginx
```sh
sudo nano /etc/nginx/sites-available/new-site.com
```

#### 8. Configure the Nginx vhost
```sh
server {
    listen 80;

    root /var/www/vhosts/new-site.com/web;
    index index.php index.html index.htm;

    server_name www.new-site.com new-site.com;

    include /etc/nginx/conf.d/server/1-common.conf;

    access_log /var/www/vhosts/new-site.com/logs/access.log;
    error_log /var/www/vhosts/new-site.com/logs/error.log warn;

    location ~ \.php$ {
        try_files $uri $uri/ /index.php?$args;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/var/run/php/php7.1-fpm-new-site.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

#### 9. Enable the new Nginx vhost
```
cd /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/new-site.com new-site.com
sudo service nginx restart ; sudo systemctl status nginx.service
```

#### 10. Configure the Apache2 vhost
```sh
<VirtualHost *:80>
    ServerAdmin admin@gmail.com
    ServerName new-site.com
    ServerAlias www.new-site.com

    DocumentRoot /var/www/vhost/new-site.com/
    <Directory />
        Options FollowSymLinks
        AllowOverride None
    </Directory>
    <Directory /var/www/vhost/new-site.com/>
        Options Indexes FollowSymLinks MultiViews
        AllowOverride All
        Order allow,deny
        Allow from All
        Require all granted
    </Directory>
</VirtualHost>
```

#### 11. Enable the new Apache2 vhost
```
cd /etc/apache2/sites-enabled/
sudo ln -s /etc/apache2/sites-available/new-site.com new-site.com
sudo service apache2 restart ; sudo systemctl status nginx.service apache2
```

### 12. MariaDB (MySQL)
```sh
sudo mysql -u root -p
> CREATE DATABASE newwebsite_tld;
> CREATE USER 'newwebsite_tld'@'localhost' IDENTIFIED BY 'password';
> GRANT ALL PRIVILEGES ON newwebsite_tld.* TO 'newwebsite_tld'@'localhost';
> FLUSH PRIVILEGES;
```

## 13. Redis
```sh
sudo apt-get update
# tcl package, which we can use to test our binaries.
sudo apt-get install build-essential tcl
sudo apt-get install make
cd /tmp
wget -O http://download.redis.io/redis-stable.tar.gz
tar xzvf redis-3.2.9.tar.gz
cd redis-3.2.9
make && make test &&sudo make install
```
### Configure Redis
```sh
sudo mkdir /etc/redis
sudo cp /tmp/redis-stable/redis.conf /etc/redis
sudo vim /etc/redis/redis.conf
```
I assume we are install on linux distro so we need config Redis run as service of systemd.
in `redis.conf ` let find `supervised` and set `supervised systemd`
Next find the `dir` This option specifies the directory that Redis will use to dump persistent data. We need to pick a location that Redis will have write permission and that isn't viewable by normal users.
We will use the `/var/lib/redis` directory for this, which we will create in a moment:
```sh
# Note that you must specify a directory here, not a file name.
dir /var/lib/redis
```
### Create a Redis systemd Unit File
```sh
sudo vim /etc/systemd/system/redis.service
```
let put some content needed
```sh
[Unit]
Description=Redis In-Memory Data Store
After=network.target

[Service]
User=redis
Group=redis
ExecStart=/usr/local/bin/redis-server /etc/redis/redis.conf
ExecStop=/usr/local/bin/redis-cli shutdown
Restart=always

[Install]
WantedBy=multi-user.target
```
### Create the Redis User, Group and Directories
```sh
sudo adduser --system --group --no-create-home redis
sudo mkdir /var/lib/redis
sudo chown redis:redis /var/lib/redis
sudo chmod 770 /var/lib/redis
```

### Start and test Redis
```sh
sudo systemctl start redis
sudo systemctl status redis
```

### To enable Redis to start at boot
```sh
sudo systemctl enable redis
```

### Install php-redis
```sh
sudo apt-get install php7.0-redis
sudo phpenmod redis && sudo service apache2 restart
```
Or if you want to manually install php-redis

```sh
sudo git clone -b php7 https://github.com/phpredis/phpredis.git
sudo mv phpredis/ /etc/ && cd /etc/phpredis
sudo phpize && sudo ./configure && sudo make && sudo make install
# depends on your php locate, it maybe /etc/php5.6 or /etc/php7.0 or /etc/php7.1 or simply /etc/php
sudo touch /etc/php7.1/mods-available/redis.ini
sudo echo 'extension=redis.so' > /etc/php7.1/mods-available/redis.ini
sudo phpenmod redis && sudo service apache2 restart
```

### Install MongoDB
```sh
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 0C49F3730359A14518585931BC711F9BA15703C6
echo "deb [ arch=amd64,arm64 ] http://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.4 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-3.4.list
sudo apt-get update
sudo apt-get install mongodb-org
sudo systemctl start mongod
sudo systemctl status mongod
# start mongodb at boot
sudo systemctl enable mongod
```

### Install php-mongo
```sh
sudo pecl install mongodb
```
and then add `extension=mongodb.so` to your `php.ini` file

## Others

### Git
```
sudo apt-get install git
```

### Postfix (sending emails from PHP)

In case you cannot send emails from PHP and getting error (`tail /var/log/mail.log`) `Network is unreachable`, you need to switch Postfix from IPv6 to IPv6.

```
sudo apt-get install postfix
sudo nano /etc/postfix/main.cf
```

Now change the line `inet_protocols = all` to `inet_protocols = ipv4` and restart postfix by `sudo /etc/init.d/postfix restart`.

You can also check if you have opened port 25 by `netstat -nutlap | grep 25`

### Munin

#### 1. Install
`apt-get install munin-node  munin`

#### 2. Configure Munin
1. Uncomment `#host 127.0.0.1` in `/etc/munin/munin-node.conf`
1. Append following code to `/etc/munin/munin-node.conf`

```
[nginx*]
env.url http://localhost/nginx_status
```

#### 3. Configure nginx `/etc/nginx/sites-available/default`
```
sudo nano /etc/nginx/sites-available/default
# Change listen 80 default_server; to
listen 80

#Change listen [::]:80 default_server; to
listen [::]:80

# Add settings for stub status to server {}
    location /nginx_status {
        stub_status on;
        access_log off;
        allow 127.0.0.1;
        deny all;
    }

# Add setting to access stats online

    location /stats {
        allow YOUR.IP.ADDRESS;
        deny all;
        alias /var/cache/munin/www/;
    }
```

#### 4. Install [plugins](https://www.github.com/munin-monitoring/contrib/)

```
cd /usr/share/munin/plugins
sudo wget -O nginx_connection_request https://raw.github.com/munin-monitoring/contrib/master/plugins/nginx/nginx_connection_request
sudo wget -O nginx_status https://raw.github.com/munin-monitoring/contrib/master/plugins/nginx/nginx_status
sudo wget -O nginx_memory https://raw.github.com/munin-monitoring/contrib/master/plugins/nginx/nginx_memory 

sudo chmod +x nginx_request
sudo chmod +x nginx_status
sudo chmod +x nginx_memory    

sudo ln -s /usr/share/munin/plugins/nginx_request /etc/munin/plugins/nginx_request
sudo ln -s /usr/share/munin/plugins/nginx_status /etc/munin/plugins/nginx_status
sudo ln -s /usr/share/munin/plugins/nginx_memory /etc/munin/plugins/nginx_memory
```

#### Restart Munin
`sudo service munin-node restart`

### Rabbitmq

Install PHP extension
```
sudo apt-get install php-amqp
```

Install RabbitMQ

```
echo 'deb http://www.rabbitmq.com/debian/ testing main' | sudo tee /etc/apt/sources.list.d/rabbitmq.list
wget -O- https://www.rabbitmq.com/rabbitmq-release-signing-key.asc | sudo apt-key add -
sudo apt-get update
sudo apt-get install rabbitmq-server
sudo service rabbitmq-server status
sudo rabbitmq-plugins enable rabbitmq_management
sudo ufw allow 15672
sudo rabbitmqctl add_user admin ********* 
sudo rabbitmqctl set_user_tags admin administrator 
sudo rabbitmqctl set_permissions -p / admin ".*" ".*" ".*"
sudo rabbitmqctl delete_user guest
sudo service rabbitmq-server restart
```

#### Installing plugin
1. Download the `.ez` plugin to `/usr/lib/rabbitmq/lib/rabbitmq_server-{version}/plugins`
1. Enable the plugin by `sudo rabbitmq-plugins enable {plugin name}`

### Supervisor

`sudo apt-get install supervisor`

### Node.js & NPM
```
sudo apt-get install nodejs
sudo apt-get install npm
```

If you are getting error `/usr/bin/env: ‘node’: No such file or directory` run
```
sudo ln -s /usr/bin/nodejs /usr/bin/node
```

### Install NodeJS via NVM
```sh
curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.33.2/install.sh | bash
# Close terminal or open another terminal and then continue execute below scripts
# Install latest LTS version of node
nvm install --lts
# Use LTS version as default
nvm use --lts
# current latest LTS is 6.11.0
nvm alias default 6.11.0
```
### Install Composer
```sh
curl -sS https://getcomposer.org/installer | sudo php -- --install-dir=/usr/local/bin --filename=composer
```

## Todo

- [ ] better description of nginx configuration
- [x] php-fpm settings
- [x] munin
- [x] script for creating new vhost
- [x] git
- [x] composer
- [x] NPM and NVM
- [ ] automysqlbackup
- [x] postfix
- [ ] automyfilebackup
- [ ] files permissions for some PHP framework like Laravel or Symfony

## Reference
- https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-14-04
- https://www.digitalocean.com/community/tutorials/how-to-install-linux-nginx-mysql-php-lemp-stack-on-ubuntu-14-04
- https://www.digitalocean.com/community/tutorials/how-to-optimize-nginx-configuration
- https://gist.github.com/jsifalda/3331643
- http://serverfault.com/questions/627903/is-the-php-option-cgi-fix-pathinfo-really-dangerous-with-nginx-php-fpm
- https://easyengine.io/tutorials/nginx/tweaking-fastcgi-buffers/
- https://gist.github.com/magnetikonline/11312172
- https://www.digitalocean.com/community/questions/warning-your-environment-specifies-an-invalid-locale-this-can-affect-your-user-experience-significantly-including-the-ability-to-manage-packages
- https://www.digitalocean.com/community/tutorials/how-to-host-multiple-websites-securely-with-nginx-and-php-fpm-on-ubuntu-14-04
- https://www.digitalocean.com/community/tutorials/how-to-create-a-new-user-and-grant-permissions-in-mysql
- https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-12-04
- https://www.digitalocean.com/community/tutorials/how-to-install-node-js-on-an-ubuntu-14-04-server
- http://stackoverflow.com/questions/21491996/installing-bower-on-ubuntu-
- http://ithelpblog.com/itapplications/howto-fix-postfixsmtp-network-is-unreachable-error/
- https://www.digitalocean.com/community/tutorials/how-to-create-hot-backups-of-mysql-databases-with-percona-xtrabackup-on-ubuntu-14-04
- https://github.com/jnstq/munin-nginx-ubuntu

### Setting PHP-FPM
 - https://www.if-not-true-then-false.com/2011/nginx-and-php-fpm-configuration-and-optimizing-tips-and-tricks/
 - http://myshell.co.uk/blog/2012/07/adjusting-child-processes-for-php-fpm-nginx/
 - https://jeremymarc.github.io/2013/04/22/nginx-and-php-fpm-for-performance
 - http://myshell.co.uk/blog/2012/07/adjusting-child-processes-for-php-fpm-nginx/
 - https://serversforhackers.com/video/php-fpm-process-management
 - https://overloaded.io/finding-process-memory-usage-linux

## License

This work is licensed under a [Creative Commons Attribution-ShareAlike 4.0 International License](http://creativecommons.org/licenses/by-sa/4.0/).
