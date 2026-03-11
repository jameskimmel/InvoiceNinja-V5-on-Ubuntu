# This is a basic tutorial for a default installation of a InvoiceNinja instance on Ubuntu 24.04.3 LTS

update and upgrade 
```bash
sudo apt update && sudo apt upgrade -y
```

## install PHP  
At the time of writing this, Ubuntu 22.04.3 uses PHP 8.3, which is the supported PHP version for InvoiceNinja.

We install php
```bash
sudo apt install php-fpm -y
```

and some php extensions
```bash
sudo apt install php-{bcmath,mbstring,xml,curl,zip,gmp,gd,mysql} -y
```

check PHP. Should show PHP8.3.6
```bash
php -v
```

To increase memory limit of php open the file
```bash
sudo nano /etc/php/8.3/fpm/php.ini 
```
press CTRL + W to search for "memory_limit"
and change it to 1 or 2 GB. Should look like this:
```bash
memory_limit = 2G
```
press CTRL + X and save it.

## install other dependencies  
```bash
sudo apt install mariadb-server curl git nginx composer -y
```
Now you can open the IP of your host and see the NGINX welcome page. Something like http://192.168.1.10 or http://localhost if you are already on the machine.
HTTPS like http://192.168.1.10 will not work yet! You need to use http instead of https!

make sure there is no Apache2 running
```bash
sudo systemctl stop apache2
sudo systemctl disable apache2
```
you should see "failed" since they do not exist.

Enable nginx at boot and start it
```bash
sudo systemctl enable --now nginx
```

Delete the nginx default site and reload. The welcome page should now be gone
```bash
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -s reload
```

Check if php-fpm is running. You should see something like Active: active (running)
```bash
sudo systemctl status php8.3-fpm.service
```

## Database
enable mariadb and start it 
```bash
sudo systemctl enable --now mariadb
```
This command will take you through a guided wizard to initialize the SQL database. Use default values written in capital letters
```bash
sudo mysql_secure_installation
```
Enter, 
Enter, 
Enter,
insert a password for the root account, 
repeat the password, 
Enter, 
Enter, 
Enter, 
Enter. 
You should see "Thanks for using MariaDB"

login to the database
```bash
sudo mysql -u root -p
```
enter the password you just set

you should see MariaDB [(none)]> on the left of your cursor

create database ninjadb
```mysql
CREATE SCHEMA `ninjadb` DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci;
```
You should see Query OK, 1 row affected

create the user ninja and set a Password
```mysql
CREATE USER 'ninja'@'localhost' IDENTIFIED BY 'Password';
```
You should see Query OK, 0 rows affected

give all permissions for to the local ninja user to access ninjadb
```mysql
GRANT ALL PRIVILEGES ON ninjadb.* TO 'ninja'@'localhost';
```
You should see Query OK, 0 rows affected

reload privileges
```mysql
FLUSH PRIVILEGES;
```
You should see Query OK, 0 rows affected

exit the db
```mysql
exit
```
You should see "Bye"

## NGINX
create config for webpage
```bash
sudo nano /etc/nginx/sites-available/invoiceninja.conf
```
insert this:
```NGINX
server {
        listen 80 default_server;
        server_name _;

        root /usr/share/nginx/invoiceninja/public;
        index index.html index.htm index.php;

	charset utf-8;
	client_max_body_size 100M;



        location / {
                # First attempt to serve request as file, then
                # as directory, then fall back to displaying a 404.
                try_files $uri $uri/ =404;
        }

        # pass PHP scripts to FastCGI server
        location ~ \.php$ {
               include snippets/fastcgi-php.conf;
               fastcgi_pass unix:/run/php/php-fpm.sock;
        }

	location = /favicon.ico { 
		access_log off; log_not_found off; 
	}

	location = /robots.txt {
		access_log off; log_not_found off; 
	}

}

```
save and exit

This config example listenes to all server names (because it will be put behind a proxy) but you can set a
server name instead of "_". For example you could use ninja.yourdomain.com

Check if syntax of your file is ok
```bash
sudo nginx -t
```

## Optional: Install SnapPDF
If you wanna use SnapPDF instead of the cloud service phantomPDF, we need some some dependacies. 
More info here https://github.com/beganovich/snappdf#headless-chrome-doesnt-launch-on-unix

```bash
sudo apt install -y \
  ca-certificates \
  fonts-liberation \
  libappindicator3-1 \
  libasound2t64 \
  libatk-bridge2.0-0t64 \
  libatk1.0-0t64 \
  libc6 \
  libcairo2 \
  libcups2t64 \
  libdbus-1-3 \
  libexpat1 \
  libfontconfig1 \
  libgbm1 \
  libgcc-s1 \
  libglib2.0-0t64 \
  libgtk-3-0t64 \
  libnspr4 \
  libnss3 \
  libpango-1.0-0 \
  libpangocairo-1.0-0 \
  libstdc++6 \
  libx11-6 \
  libx11-xcb1 \
  libxcb1 \
  libxcomposite1 \
  libxcursor1 \
  libxdamage1 \
  libxext6 \
  libxfixes3 \
  libxi6 \
  libxrandr2 \
  libxrender1 \
  libxss1 \
  libxtst6 \
  lsb-release \
  wget \
  xdg-utils \
  libgbm-dev \
  libxshmfence-dev
```

## Install invoiceNinja
make install dir
```bash
sudo mkdir /usr/share/nginx/invoiceninja && cd /usr/share/nginx/invoiceninja
```
Find latest version here https://github.com/invoiceninja/invoiceninja/releases/latest
Right click the link to the Source Code (zip) file and copy the download link

Download the file
```bash
sudo wget https://github.com/invoiceninja/invoiceninja/archive/refs/tags/v5.12.53.tar.gz
```
extract and delete it
```bash
sudo tar -xvf invoiceninja.tar.gz && sudo rm invoiceninja.tar.gz 
```

Copy the example .env 
```bash
sudo cp /usr/share/nginx/invoiceninja/.env.example /usr/share/nginx/invoiceninja/.env
```
Now we need to edit your .env file.
For now, you only need to edit one line (or two if you have a reverse proxy like me) 
Everything else we can setup later in the webgui setup. I added some hashtag comments above the line(s) you need to change.

```bash
sudo nano /usr/share/nginx/invoiceninja/.env
```

```bash
APP_NAME="Invoice Ninja"
APP_ENV=production
APP_KEY=base64:RR++yx2rJ9kdxbdh3+AmbHLDQu+Q76i++co9Y8ybbno=
APP_DEBUG=false

APP_URL=http://localhost
REACT_URL=http://localhost:3001

DB_CONNECTION=mysql
MULTI_DB_ENABLED=false

DB_HOST=localhost
DB_DATABASE=ninja
DB_USERNAME=ninja
DB_PASSWORD=ninja
DB_PORT=3306

DEMO_MODE=false

BROADCAST_DRIVER=log
LOG_CHANNEL=stack
CACHE_DRIVER=file
QUEUE_CONNECTION=sync
SESSION_DRIVER=file
SESSION_LIFETIME=120

MAIL_MAILER=smtp
REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379

POSTMARK_API_TOKEN=
REQUIRE_HTTPS=false

GOOGLE_MAPS_API_KEY=
ERROR_EMAIL=
# I set this to the IPv4 of my reverse proxy.
TRUSTED_PROXIES=10.0.25.10
SCOUT_DRIVER=null

NINJA_ENVIRONMENT=selfhost

# this you need to set to snappdf
#options - snappdf / phantom / hosted_ninja
PDF_GENERATOR=snappdf

PHANTOMJS_KEY='a-demo-key-with-low-quota-per-ip-address'
PHANTOMJS_SECRET=secret

UPDATE_SECRET=secret

DELETE_PDF_DAYS=60
DELETE_BACKUP_DAYS=60

COMPOSER_AUTH='{"github-oauth": {"github.com": "${{ secrets.GITHUB_TOKEN }}"}}'

GOOGLE_PLAY_PACKAGE_NAME=
APPSTORE_PASSWORD=

MICROSOFT_CLIENT_ID=
MICROSOFT_CLIENT_SECRET=
MICROSOFT_REDIRECT_URI=

APPLE_CLIENT_ID=
APPLE_CLIENT_SECRET=
APPLE_REDIRECT_URI=

NORDIGEN_SECRET_ID=
NORDIGEN_SECRET_KEY=

GOCARDLESS_CLIENT_ID=
GOCARDLESS_CLIENT_SECRET=

OPENEXCHANGE_APP_ID=
```
save and exit

set permissions to the nginx user
```bash
sudo chown -R www-data:www-data /usr/share/nginx/invoiceninja
```
edit crontab to run Laravel Scheduler every minute
```bash
sudo -u www-data crontab -e
```
insert this at the end:
```bash
* * * * * cd /usr/share/nginx/invoiceninja/ && php artisan schedule:run >> /dev/null 2>&1
```
save and exit

enable webpage
```bash
sudo ln -s /etc/nginx/sites-available/invoiceninja.conf /etc/nginx/sites-enabled/
```

Check if everything is good
```bash
sudo nginx -t
```
reload nginx
```bash
sudo nginx -s reload
```


The next steps are very much dependant if you use InvoiceNinja behind a proxy or not or if you only use invoiceninja local without a cert.  

If you don't use https, you don't have to do anything.  I really can't recommend it though. 

If you don't use it behind a proxy, you need to install certbot on the InvoiceNinja host. See option 2.

If you use it behind a proxy, you configure certbot on the proxy.
See how to below.  See option 3.

## Option 2: Installing certbot to get a valid cert  

install snapd  
```bash
sudo apt install snapd
```
update snapd
```bash
sudo snap install core
sudo snap refresh core
```
install certbot
```bash
sudo snap install --classic certbot
```
create a cert
```bash
sudo certbot --nginx
```
this will lead you trough the setup process. If something fails, it is probably because you forgot to set the public dns or your firewall blocks port 80 to certbot
Anyway, there are a lot of good tutorials online how to setup certbot.
Certbot automatically changes your NGINX .conf file to listen on port 443

## Option 3: running behing a proxy
if you run invoiceninja behind a proxy, here is a sample config.

```bash
server {
    server_name ninja.domain.com;

    listen       80;
    listen      [::]:80;

    # This is important or you will get 413 Server error from the proxy side
    client_max_body_size 100M;

    location / {
                    proxy_set_header        Host $host;
                    proxy_set_header        X-Real-IP $remote_addr;
                    proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
                    proxy_set_header        X-Forwarded-Proto $scheme;
                    proxy_buffering         off;
                    proxy_request_buffering off;
                    proxy_pass_header       Server;
                    # Enter the IP or Internal DNS name of the server Invoice Ninja is installed on
                    proxy_pass              http://10.0.1.2/;
                    proxy_redirect          off;
                    }
}
```



Visit our http://ipofyourhost/setup address to setup InvoiceNinja  

For URL you set the desired URL or the IP address if you use only local without SSL.    
HTTPS should be require unless you use a reverse proxy.  
Test PDF should show success.  

Insert the database credentials  
```bash
localhost
3306
ninjadb
ninja
```
and the password you set during the DB setup. If you just copied my steps, this would be "Password".    

Test PDF should show success.  

Create a User Account, agree to the terms and click submit  
Now this could take some time. You should be redirected to the URL you defined earlier.  
Don't leave the page and have some patience. If the page is just gray, try to disable pihole or any other adblockers.  

If your browser redirected you to https, you need to setup certbot first before you try to login.  

This is it. You are done and hopefully everything is up and running! You can follow the optional steps below if you wan't,
but a basic local none encrypted version sould be up and running now. 



## Optional: optimize the artisan queque
For better performance you can install supervisor.
More info here: https://invoiceninja.github.io/en/self-host-installation/#add-the-cron-job

Disable the old crontab by deleting it. 
```bash
sudo -u www-data crontab -e
```


Install and configure supervisor
```bash
sudo apt-get install supervisor  
cd /etc/supervisor/conf.d  
cd /etc/supervisor/conf.d  
sudo nano invoiceninja-worker.conf
```
insert this
```bash
[program:invoiceninja-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /usr/share/nginx/invoiceninja/artisan queue:work --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=www-data
numprocs=8
redirect_stderr=true
stdout_logfile=/var/log/invoiceninja-worker.log
stopwaitsecs=3600
```
Then do
```bash
cd /var/log
sudo touch invoiceninja-worker.log
sudo chown www-data:www-data invoiceninja-worker.log
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start invoiceninja-worker:*
sudo supervisorctl status
sudo nano /usr/share/nginx/invoiceninja/.env
```
insert this
```bash
QUEUE_CONNECTION=database

cd /usr/share/nginx/invoiceninja/
sudo -u www-data php artisan optimize
sudo -u www-data php artisan queue:restart
```
Done :)
