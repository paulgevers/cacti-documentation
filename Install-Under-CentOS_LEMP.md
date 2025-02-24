# Installing on CentOS 7

## LEMP (Linux, Nginx, MySQL, PHP) Required packages

### Web Server

1. Install Nginx

    ```console
    yum install -y nginx
    ```

2. Enable and start the service to ensure it starts when the system reboots

    ```console
    systemctl start nginx
    systemctl enable nginx
    ```

### Example configuration file for Nginx and SSL

This example assumes you have your own certificates already.  You should make
sure you change the paths to match your setup.

Replace `YourOwnCertFile.crt` and `YourOwnCertFile.key` with the names of the
files holding your certificate (`.crt`) and private key (`.key`).

```console
/etc/nginx/conf.d/cacti.conf
```

```console
# Advanced config for NGINX
#server_tokens off;
add_header X-XSS-Protection "1; mode=block";
add_header X-Content-Type-Options nosniff;

# Redirect all HTTP traffic to HTTPS
server {
   listen 80;
   server_name cacti.yourdomain.com; #No one likes unencrypted web servers
   #return 301 https://$host$request_uri; # some nginx do not support 'return';
}

# SSL configuration
server {
   listen 443 ssl default deferred;
   server_name cacti.yourdomain.com;
   root /usr/share/nginx/html/cacti;
   index index.php index.html index.htm;

   # Compression increases performance0
   gzip on;
   gzip_types      text/plain text/html text/xml text/css application/xml application/javascript application/x-javascript application/rss+xml application/xhtml+xml;
   gzip_proxied    no-cache no-store private expired auth;
   gzip_min_length 1000;

   location / {
      try_files $uri $uri/ /index.php$query_string;
   }

   error_page 404 /404.html;
   error_page 500 502 503 504 /50x.html;
   location = /50x.html {
      root /usr/share/nginx/html/;
   }

   location ~ \.php$ {
      alias /usr/share/nginx/html/cacti;
      index index.php
      try_files $uri $uri/ =404;
      fastcgi_split_path_info ^(.+\.php)(/.+)$;

      # you may have to change the path here for your OS
      fastcgi_pass unix:/var/run/php-fpm/php-fpm.sock;
      fastcgi_index index.php;
      fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
      include /etc/nginx/fastcgi_params;
   }

   location /cacti {
      root /usr/share/nginx/html/;
      index index.php index.html index.htm;
      location ~ ^/cacti/(.+\.php)$ {
         try_files $uri =404;
         root /usr/share/nginx/html;

         # you may have to change the path here for your OS
         fastcgi_pass unix:/var/run/php-fpm/php-fpm.sock;
         fastcgi_index index.php;
         fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
         include /etc/nginx/fastcgi_params;
      }

      location ~* ^/cacti/(.+\.(jpg|jpeg|gif|css|png|js|ico|html|xml|txt))$ {
         expires max;
         log_not_found off;
      }
   }

   location /doc/ {
      alias /usr/share/nginx/html/cacti/doc/;
      location ~* ^/docs/(.+\.(html|md|txt))$ {
         root /usr/share/nginx/html/cacti/;
         autoindex on;
         allow 127.0.0.1; # Change this to allow your local networks
         allow ::1;
         deny all;
      }
   }

   location /cacti/rra/ {
      deny all;
   }

   ## Access and error logs.
   access_log /var/log/nginx/cacti_access.log;
   error_log  /var/log/nginx/cacti_error.log info;

   ssl_certificate      /etc/ssl/certs/YourOwnCertFile.crt;
   ssl_certificate_key  /etc/ssl/private/YourOwnCertKey.key;

   # Improve HTTPS performance with session resumption
   ssl_session_cache shared:SSL:10m;
   ssl_session_timeout 5m;

   # Enable server-side protection against BEAST attacks
   #ssl_prefer_server_ciphers on;
   ssl_ciphers ECDH+AESGCM:ECDH+AES256:ECDH+AES128:DH+3DES:!ADH:!AECDH:!MD5;

   # Disable SSLv3
   ssl_protocols TLSv1 TLSv1.1 TLSv1.2;

   # Diffie-Hellman parameter for DHE cipher suites
   # $ sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 4096
   ssl_dhparam /etc/ssl/certs/dhparam.pem;

   # Enable HSTS (https://developer.mozilla.org/en-US/docs/Security/HTTP_Strict_Transport_Security)
   add_header Strict-Transport-Security "max-age=63072000; includeSubdomains";
}
```

### Database Server

The choice between MySQL server and MariaDB is normally down to the OS
maintainers if you use one of the predefined LAMP setup installations.  If you
are deciding between these yourself, you should research this via your
favorite search engine.

Whilst MySQL is the original open source SQL database server created in 1995
which is now owned by Oracle, MariaDB is designed as a drop-in replacement by
some of the original MySQL developers / owners.  This will likely continue to
be a drop-in alternative until there is a major divergence that can not be
bridged.

#### MySQL

1. Install MySQL server

    ```console
    yum install -y mysql mysql-server
    ```

2. Enable and start the service to ensure it starts when the system reboots

    ```console
    systemctl enable mysqld
    systemctl start mysqld
    ```

#### MariaDB

1. Install MariaDB server

   ```console
   yum install -y MariaDB-server MariaDB-client
   ```

2. Enable and start the service to ensure it starts when the system reboots

    ```console
    systemctl enable mariadb
    systemctl start mariadb
    ```

### MySQL/MariaDB common tasks and recommendations

**IMPORTANT**: Secure your MySQL installation before doing any more changes

```console
/usr/bin/mysql_secure_installation
```

The following MySQL/MariaDB recommendations may vary depending on your system
setup. In any case, Cacti will prompt you with more accurate recommendations
during the installation.

1. Edit your **server.cnf** file

    ```console
    vim /etc/my.cnf.d/server.cnf
    ```

    The following `[mysqld]` section is a base configuration.  The installer
will provide recommendations based on the actual system which will be more
tailored to your environment.

    ```shell
    [mysqld]
    character-set-server=utf8mb4
    collation-server=utf8mb4_unicode_ci
    max_allowed_packet=18M
    max_heap_table_size=98M
    tmp_table_size=64M
    join_buffer_size=64M
    innodb_buffer_pool_size=488M
    innodb_doublewrite=OFF
    innodb_flush_log_at_timeout=3
    innodb_read_io_threads=32
    innodb_write_io_threads=16
    log-error                      = /var/log/mysql/mysql-error.log
    log-queries-not-using-indexes  = 1
    slow-query-log                 = 1
    slow-query-log-file            = /var/log/mysql/mysql-slow.log
    ```

2. Restart MySQL/MariaDB service to pick up the changes

    ```console
    systemctl restart mysql
    ```

3. Populate timezone table with available timezones

    ```console
    mysql_tzinfo_to_sql /usr/share/zoneinfo | mysql -u root -p mysql
    ```

#### Setup Cacti database

1. Login to MySQL/MariaDB as root to create Cacti database

    ```console
    # mysql -u root -p
    MariaDB [(none)]> create database if not exists cacti;
    Query OK, 1 row affected (0.00 sec)
    ```

2. Import Cacti database from SQL file

   ```sql
   MariaDB [(none)]> use cacti;
   Database changed
   MariaDB [(cacti)]> source /var/www/html/cacti/cacti.sql
   ```

3. Grant Cacti username access to Cacti database. Replace `your_cacti_username`
 and `your_cacti_password` with your own details.

    ```sql
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON cacti.* TO 'your_cacti_username'@'localhost' IDENTIFIED BY
    'your_cacti_password';
    Query OK, 0 rows affected (0.00 sec)
    ```

4. Grant cacti username to MySQL timezone table

    ```sql
    MariaDB [(none)]> GRANT SELECT ON mysql.time_zone_name TO 'cacti'@'localhost';
    Query OK, 0 rows affected (0.00 sec)
    MariaDB [(none)]> FLUSH PRIVILEGES;
    Query OK, 0 rows affected (0.00 sec)
    ```
    
5. Save the Database Charset and Collation

   ```sql
   MariaDB [(none)]> ALTER DATABASE cacti CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
   MariaDB [(none)]> FLUSH PRIVILEGES;
   ```

### Common PHP packages

#### PHP

PHP and various packages are all required by Cacti for successful operation

1. Install PHP and required packages.

    ```console
    yum install -y php php-common php-bcmath php-cli \
    php-mysqlnd php-gd php-gmp php-intl \
    php-json php-ldap php-mbstring \
    php-pdo php-pear php-snmp php-process \
    php-xml php-zip php-fpm
    ```

---
Note: **php-fpm** is only required if your Web Server is Nginx

2. Set a timezone to your PHP.INI configuration

    Edit php.ini, typically located at `/etc/php.ini`

    ```console
    date.timezone = Pacific/Auckland
    ```

3. Disable insecure `cgi.fix_pathinfo`

    ```console
    cgi.fix_pathinfo=0
    ```

#### Setup php-fpm

1. Enable and start the service to ensure it starts when the system reboots

    ```console
    systemctl start php-fpm
    systemctl enable php-fpm
    ```

2. Edit `/etc/php-fpm.d/www.conf`

    Find `listen = 127.0.0.1:9000` and add the following line below

    ```console
    listen = /var/run/php-fpm/php-fpm.sock
    ```

    Find `listen.owner` and `listen.group` and set them to nginx

    ```console
    listen.owner = nginx
    listen.group = nginx
    ```

    Find user and group and set them to `nginx`

    ```console
    user = nginx
    group = nginx
    ```

    Restart php-fpm to pick up the changes

    ```console
    systemctl restart php-fpm
    ```

### RRDtool

RRDtool is required to store the data retrieved from devices in `.rra` files to
produce the graphs which are shown within Cacti

```console
yum install -y rrdtool
```

### SNMP

SNMP is used to query most devices for information.

```console
yum install -y net-snmp net-snmp-utils
echo "rocommunity public" > /etc/snmp/snmpd.conf
systemctl enable snmpd
systemctl start snmpd
```

### Cacti

The following steps will show you how to manually download, install and
configure the basics for Cacti.

1. Download Cacti source code from [Cacti Web
 Site](https://www.cacti.net/download_cacti.php)

   ```console
   cd /tmp
   wget https://www.cacti.net/downloads/cacti-1.y.z.tar.gz
   tar -zxvf cacti-1.y.z.tar.gz
   mv -v cacti-1.y.z /usr/share/nginx/html/cacti
   ```

2. Edit the config.php file

    ```console
    mv -v /usr/share/nginx/html/cacti/include/config.php-dist /usr/share/nginx/html/cacti/include/config.php
    ```

3. Update database_ fields with your own details. This section only applies to
 the Main Cacti Server

    ```php
    $database_type     = 'mysql';
    $database_default  = 'your_cacti_database';
    $database_hostname = 'localhost';
    $database_username = 'your_cacti_username';
    $database_password = 'your_cacti_password';
    $database_port     = '3306';
    $database_ssl      = false;
    $database_ssl_key  = '';
    $database_ssl_cert = '';
    $database_ssl_ca   = '';
    ```

4. Set your cookie domain to match your web site domain name

    ```console
    $cacti_cookie_domain = 'cacti.yourdomain.com';
    ```

5. Create your cron task file or systemd units file

   Starting with Cacti 1.2.16, you have the option to use either the
   legacy Crontab entry, or an optional cactid units file and server
   to run your Cacti pollers.

   For Crontab use, follow the instructions below:

   Create and edit `/etc/cron.d/cacti` file.
   Make sure to setup the correct path to poller.php

   ```console
   */5 * * * * nginx php /var/www/html/cacti/poller.php &>/dev/null
   ```

   For systemd unit's file install, you will need to modify the
   included units file to following your install location
   and desired user and group's to run the Cacti poller as.
   To complete the task, follow the procedure below:

   ```console
   vim /var/www/html/cacti/service/cactid.service (edit the path)
   touch /etc/sysconfig/cactid
   cp -p /var/www/html/cacti/service/cactid.service /etc/systemd/system
   systemctl enable cactid
   systemctl start cactid
   systemctl status cactid
   ```

   The systemd units file makes managing a highly available Cacti
   setup a bit more convenient.

#### Spine

1. Install the necessary packages to compile and install spine

   ```console
   yum install -y autoconf automake libtool dos2unix help2man \
   openssl-devel mariadb-devel net-snmp-devel
   ```

2. Download spine source code from [Cacti Web
 Site](https://www.cacti.net/spine_download.php)

   Go to /tmp to download the source code and extract it

   ```console
   cd /tmp
   wget https://www.cacti.net/downloads/spine/cacti-spine-1.y.z.tar.gz
   tar -zxvf cacti-spine-1.y.z.tar.gz
   cd cacti-spine-1.y.z
   ```

3. Run the configure script and compile spine.

   ```console
   # ./configure
   # make &  make install
   config/install-sh -c -d '/usr/local/spine/bin'
   /bin/sh ./libtool   --mode=install /usr/bin/install -c spine '/usr/local/spine/bin'
   libtool: install: /usr/bin/install -c spine /usr/local/spine/bin/spine
 config/install-sh -c -d '/usr/local/spine/etc'
   /usr/bin/install -c -m 644 spine.conf.dist '/usr/local/spine/etc'
   config/install-sh -c -d '/usr/local/spine/share/man/man1'
 /usr/bin/install -c -m 644 spine.1 '/usr/local/spine/share/man/man1'
   ```

4. Edit spine.conf

   Rename spine.conf.dist to spine.conf

   ```console
   mv -v /usr/local/spine/etc/spine.conf.dist /usr/local/spine/etc/spine.conf
   vi /usr/local/spine/etc/spine.conf
   ```

5. Now set up your database connection

   ```console
   DB_Host       localhost
   DB_Database   your_cacti_database
   DB_User       your_cacti_username
   DB_Pass       your_cacti_password
   DB_Port       3306
   #DB_UseSSL    0
   #RDB_SSL_Key
   #RDB_SSL_Cert
   #RDB_SSL_CA
   ```

### Security Enhanced Linux (SELinux)

If you are having issues to access the web page, disable SELinux temporarily to
prove that the issues come from the SELinux policy. It is NOT recommended to
disable SELinux permanently.

[CentOS](https:////wiki.centos.org/es/HowTos/SELinux) has a lot of
documentation on how to make your SELinux policy right.

Check SELinux status

```console
getenforce
```

Disable SELinux temporarily

```console
setenforce 0
```

Enable SELinux back

```console
setenforce 1
```

**Note:** If you installed Cacti out of `/usr/share/nginx/html` make sure you  fix
up all SELinux context and permissions.

---
<copy>Copyright (c) 2004-2023 The Cacti Group</copy>
