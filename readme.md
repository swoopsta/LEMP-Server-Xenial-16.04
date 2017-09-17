## **LEMP Server on Ubuntu 16.04 Xenial**
### Nginx, PHP7, MariaDB 10, FastCGI Cache, Brotli, HTTP2, and CloudFlare SSL

This step by step tutorial takes you through an advanced LEMP stack install for hosting WordPress sites. Building a server using Nginx, PHP 7, and MariaDB makes WordPress run faster than other options, so configuring a setup like this will usually get you the most bang for your hosting buck. We'll be utilizing FastCGI Cache, a webserver method of file caching which is built right into Nginx. By using FastCGI Cache, instead of WordPress caching plugins like W3 Total Cache or WP Super Cache, we're moving the job of caching pages to Nginx instead of PHP. This results in a faster site that can handle larger spikes in traffic. We'll also be self-signing an SSL certificate on the server-side since we're going to be using a free SSL certificate issued by CloudFlare.

----------

### **Basics**
##### **Initial Setup**
```
sudo apt-get update && sudo apt-get upgrade -y && sudo apt-get dist-upgrade -y
sudo apt-get install autotools-dev build-essential checkinstall curl debhelper dh-systemd gcc git htop libbz2-dev libexpat-dev libgd2-noxpm-dev libgd2-xpm-dev libgeoip-dev libgoogle-perftools-dev libluajit-5.1-dev libmhash-dev libpam0g-dev libpcre3 libpcre3-dev libperl-dev libssl-dev libxslt1-dev make nano openssl po-debconf software-properties-common sudo tar unzip wget zlib1g zlib1g-dbg zlib1g-dev -y
sudo locale-gen en_US.UTF-8
export LANG=en_US.UTF-8
```
##### **Removing Stuff We Don't Need**
```
sudo apt-get remove --purge mysql-server mysql-client mysql-common apache2* php5* nginx -y
sudo rm -rf /var/lib/mysql
sudo apt-get autoremove -y && sudo apt-get autoclean -y
```
##### **Changing SSH Port**
We like to add a tiny bit of extra security by changing the default SSH port. This is not a replacement for a firewall.

Change port 22 to whatever number you'd like.
```
nano /etc/ssh/sshd_config
service ssh restart
```

----------

### **Nginx**
We prefer using the **Mainline** version of Nginx instead of the **Stable** version. Mainline tends to have improved bleeding edge feature support, but this can lead to some instability. If you want Stable, change the version in the code below to whatever the latest Stable release is. We're going to be compiling Nginx from source since we want to run some custom modules and use the latest version of OpenSSL for HHTP2 support.

##### **Downloading Nginx**
First we'll need to download the latest versions of Nginx and the various modules we're using.
You'll want to check their sites to ensure you're downloading the latest version.
Get the latest versions at: [Nginx](http://nginx.org/en/download.html), [OpenSSL](https://www.openssl.org/source/), [Headers More Module](https://github.com/openresty/headers-more-nginx-module/tags), and [Nginx Cache Purge Module](http://labs.frickle.com/nginx_ngx_cache_purge/).
```
cd /usr/src/
wget http://nginx.org/download/nginx-1.13.5.tar.gz
tar -xzvf nginx-1.13.5.tar.gz
wget https://github.com/openresty/headers-more-nginx-module/archive/v0.32.tar.gz
tar -xzf v0.32.tar.gz
wget http://labs.frickle.com/files/ngx_cache_purge-2.3.tar.gz
tar -xzf ngx_cache_purge-2.3.tar.gz
wget https://www.openssl.org/source/openssl-1.1.0f.tar.gz
tar -xzf openssl-1.1.0f.tar.gz
```

##### **Brotli Compression** 
We're adding in support for Brotli compression. Brotli is Google's new lossless compression format. Brotli will take priority over gzip when enabled. Check to make sure your CDN actually works with Brotli, it may just normalize it to use gzip. If and when your CDN supports Brotli, yyour site will be ready to take advantage of this.
```
cd /usr/src
git clone https://github.com/google/ngx_brotli.git
cd ngx_brotli
git submodule update --init --recursive

```

##### **Installing Nginx**
Now it's time to compile Nginx using the parts we've downloaded. If you're running version numbers that difer from the versions we had listed above, don't forget to change the OpenSSL, Nginx Cache Purge, and Nginx More Headers module versions inside of the `./configure` command below. 
```
cd /usr/src/nginx-1.13.5
./configure --prefix=/usr/local/nginx --sbin-path=/usr/sbin/nginx --conf-path=/etc/nginx/nginx.conf --pid-path=/var/run/nginx.pid --lock-path=/var/lock/nginx.lock --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --http-fastcgi-temp-path=/var/lib/nginx/fastcgi --user=www-data --group=www-data --without-mail_pop3_module --with-openssl=/usr/src/openssl-1.1.0f --without-mail_imap_module --without-mail_smtp_module --without-http_uwsgi_module --without-http_scgi_module --without-http_memcached_module --with-http_ssl_module --with-http_stub_status_module --with-http_v2_module --with-debug --with-pcre-jit --with-http_stub_status_module --with-http_realip_module --with-http_auth_request_module --with-http_addition_module --with-http_dav_module --with-http_flv_module --with-http_geoip_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_image_filter_module --with-http_sub_module --with-http_xslt_module --with-mail --with-mail_ssl_module --with-stream --with-stream_ssl_module --with-threads --add-module=/usr/src/ngx_cache_purge-2.3 --add-module=/usr/src/headers-more-nginx-module-0.32 --add-module=/usr/src/ngx_brotli
make
sudo checkinstall
```

Using the checkinstall command tells the server to package our compiled source into a more easily managed .deb package file. Move through the prompts. You can tell it not to list the installation files, and yes to exclude them from the package. Since Nginx updates quite frequently, doing this allows us to easily upgrade later on. To upgrade to the latest version, double check Nginx and module versions (as this guide may not be up to date), then simply repeat the installation process above. Restart Nginx and you should be running the latest version.

Once again, get the latest versions at: [Nginx](http://nginx.org/en/download.html), [OpenSSL](https://www.openssl.org/source/), [Headers More Module](https://github.com/openresty/headers-more-nginx-module/tags), and [Nginx Cache Purge Module](http://labs.frickle.com/nginx_ngx_cache_purge/).

Double check that we've got everything installed correctly by using the `nginx -Vv` command. This will also list all installed modules, and your OpenSSL version.

##### **Creating Directories and Setting Permissions** 
Here we're going to ensure that the right folders are in place for our config.
```
sudo mkdir -p /var/www/html
sudo mkdir -p /var/lib/nginx/fastcgi
sudo mkdir -p /etc/nginx/ssl
sudo mkdir -p /etc/nginx/conf.d
sudo mkdir -p /var/cache/nginx
sudo mkdir -p /var/log/domains
sudo chown -hR www-data:www-data /var/log/domains
sudo rm -rf /etc/nginx/sites-enabled
sudo rm -rf /etc/nginx/sites-available
```

##### **Automatically Starting Nginx**
Now that we've installed Nginx, we'll need to make it start up automatically each time the server reboots. Ubuntu 16.04 uses SystemD to handle bootup processing, so that's what we'll be working with.
```
sudo nano /lib/systemd/system/nginx.service
```
Now paste in the code below, then save.
```
# Stop dance for nginx
# =======================
#
# ExecStop sends SIGSTOP (graceful stop) to the nginx process.
# If, after 5s (--retry QUIT/5) nginx is still running, systemd takes control
# and sends SIGTERM (fast shutdown) to the main process.
# After another 5s (TimeoutStopSec=5), and if nginx is alive, systemd sends
# SIGKILL to all the remaining processes in the process group (KillMode=mixed).
#
# nginx signals reference doc:
# http://nginx.org/en/docs/control.html
#
[Unit]
Description=A high performance web server and a reverse proxy server
After=network.target

[Service]
Type=forking
PIDFile=/run/nginx.pid
ExecStartPre=/usr/sbin/nginx -t -q -g 'daemon on; master_process on;'
ExecStart=/usr/sbin/nginx -g 'daemon on; master_process on;'
ExecReload=/usr/sbin/nginx -g 'daemon on; master_process on;' -s reload
ExecStop=-/sbin/start-stop-daemon --quiet --stop --retry QUIT/5 --pidfile /run/nginx.pid
TimeoutStopSec=5
KillMode=mixed

[Install]
WantedBy=multi-user.target
```
Finally, let's double check that it's working, and then turn on Nginx.
```
sudo systemctl enable nginx.service
sudo systemctl start nginx.service
sudo systemctl status nginx.service
```

In the future, you can restart Nginx by typing `sudo service nginx restart`.

----------

### **PHP 7**
With Nginx out of the way, it's time to install PHP 7.
```
sudo apt-get install php-fpm php-mysql php7.0-mysql php7.0-curl php7.0-gd php7.0-intl php-pear php-imagick php7.0-imap php7.0-mcrypt php-apcu php-memcache php7.0-pspell php7.0-recode php7.0-sqlite3 php7.0-tidy php7.0-xmlrpc php7.0-xsl php7.0-mbstring php-gettext
```

Now we're going to make a simple change to the **/etc/php/7.0/fpm/php.ini** file. This is a security related change, so be sure to do it.
```
sudo nano /etc/php/7.0/fpm/php.ini
```

Now find the entry for `cgi.fix_pathinfo`. Change the value from `0` to `1`. The line should read `cgi.fix_pathinfo=1`.

You may want to make some additional performance changes to PHP based on your server. If you want to change things such as how much memory is available to WordPress or how large of a file you can upload, you'll need to make these changes inside **php.ini** as well. Below are some changes that may be of use to you. Each server is different, so you'll want to alter these values based on your site's needs.
```
upload_max_filesize = 32M
post_max_size = 48M
memory_limit = 196M
max_execution_time = 600
max_input_vars = 10000
max_input_time = 400
```

Then simply restart PHP and we're done.
```
sudo service php7.0-fpm restart
```

### **MariaDB 10** 
We're using MariaDB instead of MySQL, as the performance is great with WordPress. We're running the **Stable** release of MariaDB. You can find the latest version at [https://downloads.mariadb.org/](https://downloads.mariadb.org/).

----------

##### **Add MariaDB Repo** 
```
sudo apt-key adv --recv-keys --keyserver hkp://keyserver.ubuntu.com:80 0xF1656F24C74CD1D8
sudo add-apt-repository 'deb [arch=amd64,i386,ppc64el] http://nyc2.mirrors.digitalocean.com/mariadb/repo/10.2/ubuntu xenial main'
```

##### **Installing MariaDB** 
At the end of this installation, MariaDB will ask you to set your password, don't lose this!
```	
sudo apt-get update && apt-get install mariadb-server -y
```

Make sure that MariaDB has upgraded to the latest release by running this again.
```
sudo apt-get update && sudo apt-get upgrade -y && sudo apt-get dist-upgrade -y
```

##### **Securing MariaDB** 
MariaDB includes some test users and databases that we don't want to be using in a live production environment. Now that MariaDB is installed, run this command. Since we've already set the admin password, we can hit `N` to the first option. You'll want to hit `Y` to the rest of the questions.
```
mysql_secure_installation
```

##### **Log in to MariaDB** 
Test to make sure things are working by logging into MySQL, then exiting.
```
sudo mysql -v -u root -p
```

You can exit MariaDB by typing `exit`.

----------

### **Making Things Work**
We're going to take a moment to move some files and verify that things are working.

#### **.conf Files** 
Now it's time to move [nginx.conf](https://raw.githubusercontent.com/VisiStruct/LEMP-Server-Xenial-16.04/master/nginx.conf), [wpsecurity.conf](https://raw.githubusercontent.com/VisiStruct/LEMP-Server-Xenial-16.04/master/wpsecurity.conf), and [fileheaders.conf](https://raw.githubusercontent.com/VisiStruct/LEMP-Server-Xenial-16.04/master/fileheaders.conf) into **/etc/nginx**. 
```
sudo wget https://raw.githubusercontent.com/VisiStruct/LEMP-Server-Xenial-16.04/master/nginx.conf -O /etc/nginx/nginx.conf
sudo wget https://raw.githubusercontent.com/VisiStruct/LEMP-Server-Xenial-16.04/master/wpsecurity.conf -O /etc/nginx/wpsecurity.conf
sudo wget https://raw.githubusercontent.com/VisiStruct/LEMP-Server-Xenial-16.04/master/fileheaders.conf -O /etc/nginx/fileheaders.conf
```

You'll also want to move [default.conf](https://raw.githubusercontent.com/VisiStruct/LEMP-Server-Xenial-16.04/master/conf.d/default.conf "/etc/nginx/conf.d/default.conf") into **/etc/nginx/conf.d**. 
```
sudo wget https://raw.githubusercontent.com/VisiStruct/LEMP-Server-Xenial-16.04/master/conf.d/default.conf -O /etc/nginx/conf.d/default.conf
```

Then restart PHP and Nginx.
```
sudo service nginx restart && sudo service php7.0-fpm restart
```

##### **Set Nginx Worker Processes**
Set worker processes to the number of CPUs you have available. We can find this information by using the `grep -c processor /proc/cpuinfo` command and editing the **[nginx.conf](https://raw.githubusercontent.com/VisiStruct/LEMP-Server-Xenial-16.04/master/nginx.conf)** file. Enter whatever value `grep -c processor /proc/cpuinfo` lists.
```
grep -c processor /proc/cpuinfo
sudo nano /etc/nginx/nginx.conf
```

#### **Get Your PHP Installation Info** 
Here we're going to write a very basic php file that will display the information related to our PHP installation. This lets us verify that PHP 7 is working correctly. In addition, you can use this to reference your server's PHP configuration settings in the future. We're going to send it straight to your server's default folder, which will be **/var/www/html**. By contrast, domains will be using **/var/www/yourdomain.com/html**.
```
sudo echo "<?php phpinfo(); ?>" > /var/www/html/phpinfo.php
```

Point your browser to http://ipa.ddr.ess/phpinfo.php.

----------

### **phpMyAdmin**
Since phpMyAdmin is already available through the default Ubuntu 16.04 repos, this part is really easy. We're pointing our phpMyAdmin location to **/var/www/html**, which will make it available at your server's IP address. Alter the lines below to reflect a different location, such as a behind a domain.
```
sudo apt-get install phpmyadmin -y
sudo update-rc.d -f apache2 remove
sudo ln -s /usr/share/phpmyadmin /var/www/html
```

Point your browser to http://ipa.ddr.ess/phpmyadmin.

----------

### **WordPress** 
From this point on in the tutorial, any time you see `yourdomain.com` you'll want to alter that text to whatever your domain actually is.

##### **Creating a MySQL Database** 
We're going to create the database by command line because we're cool. You can also do this directly though phpMyAdmin if you're not as cool. Replace the `database`, `user`, and `password` variables in the code below.
```
mysql -u root -p
CREATE DATABASE database;
CREATE USER 'user'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON database.* TO 'user'@'localhost';
FLUSH PRIVILEGES;
exit
```

##### **Install WordPress** 
We're going to create a few directories needed for WordPress, set the permissions, and download WordPress. We're also going to just remove the Hello Dolly plugin, because obviously.
```
sudo mkdir -p /var/www/yourdomain.com/html						
cd /var/www/yourdomain.com/html
wget http://wordpress.org/latest.tar.gz
tar -xzvf latest.tar.gz
mv wordpress/* .
rmdir /var/www/yourdomain.com/html/wordpress
sudo rm -f /var/www/yourdomain.com/html/wp-content/plugins/hello.php
sudo mkdir -p /var/www/yourdomain.com/html/wp-content/uploads
```
If you're transfering a site over, it's time to upload any files you might have (themes, plugins, uploads, etc, wp-config, etc). You'll want to do this before assigning permissions to the directories. If not, proceed to the next step.

##### **Secure WordPress** 
Once you're done uploading files, we'll want to secure WordPress' directory and file permissions.
```
find /var/www/yourdomain.com/html/ -type d -exec chmod 755 {} \;
find /var/www/yourdomain.com/html/ -type f -exec chmod 644 {} \;
sudo chown -hR www-data:www-data /var/www/yourdomain.com/html/
```

##### **Install Nginx Site File**
Now that we've got the directory structure of your domain squared away, we'll need to enable it in Nginx.

Add [yourdomain.com.conf](https://raw.githubusercontent.com/VisiStruct/LEMP-Server-Xenial-16.04/master/conf.d/yourdomain.com.conf) to **/etc/nginx/conf.d**. This folder may hold as many virtual domains as you'd like, just make a new file with a different name for each domain you want to host. Remember to change the `/etc/nginx/conf.d/yourdomain.com.conf` portion to reflect your actual domain name.
```
sudo wget https://raw.githubusercontent.com/VisiStruct/LEMP-Server-Xenial-16.04/master/conf.d/yourdomain.com.conf -O /etc/nginx/conf.d/yourdomain.com.conf
```

----------

### **Self-Signed SSL Certificate** 
Here we're going to generate a self-signed SSL certificate. Since we're using CloudFlare anyway, we're going to use a *FREE* SSL certificate through them. You'll need to set CloudFlare's SSL certificate status to `Full` for this to work.
```
sudo openssl req -x509 -nodes -days 365000 -newkey rsa:2048 -keyout /etc/nginx/ssl/yourdomain.com.key -out /etc/nginx/ssl/yourdomain.com.crt
cd /etc/nginx/ssl
openssl dhparam -out yourdomain.com.pem 2048
sudo service nginx restart && sudo service php7.0-fpm restart
```

----------

### **FastCGI Cache Conditional Purging** 

You'll want a way to purge the cache when you make changes to the site, such as editing a post, changing a menu, or deleting a comment.

##### **Nginx Cache WordPress Plugin**

We like RTCamp's Nginx Helper Plugin. You'll want to go to the WordPress Dashboard, then Settings/ Nginx Helper. Turn on purging, and select the conditions you want to trigger the purge. Finally, select the timestamp option at the bottom to display your page's build time in the source code.

Download: [Nginx Helper](https://wordpress.org/plugins/nginx-helper/)

----------

### **Checking FastCGI Cache** 
It's always a good idea to make sure that what you think is working is in fact actually working. Since we don't want to serve cached versions of every page on the site, inside [yourdomain.com.conf](https://raw.githubusercontent.com/VisiStruct/LEMP-Server-Xenial-16.04/master/conf.d/yourdomain.com.conf) we've added rules that prevent certain pagees from being cached. To help shed light on things a bit, we've added the line `add_header X-Cached $upstream_cache_status;`. This will tell us with certainty whether or not the page being served is the cached version. 

We can check the status of any page by viewing the headers that are sent along when you visit it. To do this, you can use a variety of methods. You can use the `CURL` command inside your terminal by typing `curl -I https://yourdomain.com`. Extensions exist for FireFox and Chrome that will make things a bit easier. We prefer the [HTTP Headers](https://chrome.google.com/webstore/detail/http-headers/nioieekamcpjfleokdcdifpmclkohddp "HTTP Headers for Google Chrome") Chrome extension.

You'll encounter 4 different messages based on the cache type. `X-Cached: HIT`, `X-Cached: MISS`, `X-Cached: EXPIRED`, or `X-Cached: BYPASS`. 

###### X-Cached: HIT
You're being served a cached version of the page.

###### X-Cached: MISS 
The server did not have a cached copy of that page, so you're being fed a live version instead. Initially all pages will show as `X-Cached: MISS`. Once they've been visited, Nginx will store a copy of that code for future visitors.

###### X-Cached: EXPIRED 
The version that was stored on the server is too old, and you're seeing a live version instead.

###### X-Cached: BYPASS 
We've told Nginx skip caching a page if it matches a set of criteria. For example, we don't want to cache any page beginning with `WP-`, or any page visited by a logged in user or recent commenter. Depending on the nature of your site, there may be additional things you'll want to set to avoid being cached.

----------

### **Optional Stuff** 
##### **WooCommerce and FastCGI Cache** 
We really don't want Nginx to cache anything related to WooCommerce, as this could result in a customer's information being fed to others. So we're going to tackle this two different ways. Our [yourdomain.com.conf](https://raw.githubusercontent.com/VisiStruct/LEMP-Server-Xenial-16.04/master/conf.d/yourdomain.com.conf) has the code commented out by default, as not everyone will be running woocommerce on their site.

The first method of preventing WooCommerce caching has Nginx looking for default WooCommerce pages such as Shop, Cart, or Checkout. If you've altered the names of the default WooCommerce pages, you'll also need to make those changes inside this file otherwise they will be cached.
	
The second method avoids caching by looking to see if a WooCommerce session cookie has been set. If the user has interacted with a WooCommerce page on your site, this cookie will set and caching will be disabled for that user.

Find the code below and uncomment it to enable both methods.
```
#if ($request_uri ~* "/shop.*|/cart.*|/my-account.*|/checkout.*|/addons.*") {
#		set $no_cache 1;
#	}
 
# If WooCommerce cookie is set, do not cache.
#if ($http_cookie ~* "wp_woocommerce_session_[^=]*=([^%]+)%7C") {        
#		set $no_cache 1;
#	}
```
    
If you're still caching stuff during WooCommerce sessions, there is a third method inside the file that avoids caching if an item is in the cart. It's almost never needed, but we're just making you aware that it's in there.

### **Done!** 

*Naturally, this tutorial is always subject to change, and could include mistakes or vulnerabilities that might result in damage to your site by malicious parties. We make no guarantee of the performance, and encourage you to read and thoroughly understand each setting and command before you enable it on a live production site.*

If we've helped you, or you've given up and want to hire a consultant to set this up for you, visit us at **[VisiStruct.com](https://VisiStruct.com "VisiStruct.com")**.
