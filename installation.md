# This is a basic tutorial for a default installation of a InvoiceNinja instance on Ubuntu 24.04.3 LTS

Make sure we are up to date: 
```bash
sudo apt update && sudo apt upgrade -y
```

## install PHP  
At the time of writing this, Ubuntu 22.04.3 uses PHP 8.3, which is the supported PHP version for InvoiceNinja.

We install php-fpm and some extensions:
```bash
sudo apt install php8.3-{fpm,bcmath,mbstring,xml,curl,zip,gmp,gd,mysql,intl} -y
```

To increase memory limit of php, open the ini:
```bash
sudo nano /etc/php/8.3/fpm/php.ini
```
press CTRL + W to search for "memory_limit"
and change it to 1 or 2 GB. Should look like this:
```bash
memory_limit = 2G
```
press `CTRL + X` and save it with `Y`.

Reload php-fpm to apply the change:
```bash
sudo systemctl reload php8.3-fpm.service
```

## install other dependencies  
```bash
sudo apt install mariadb-server curl git nginx composer -y
```

Enable nginx, mariadb and php8.3-fpm at boot:
```bash
sudo systemctl enable --now nginx mariadb php8.3-fpm 
```

make sure there is no Apache2 running:
```bash
sudo systemctl stop apache2 && sudo systemctl disable apache2
```
you should see "failed" since they do not exist.

Now you can open the IP of your host and see the NGINX welcome page. Something like http://192.168.1.10 or http://localhost if you are already on the machine.
HTTPS like https://192.168.1.10 will not work yet! You need to use http instead of https!

Delete the nginx default site and reload: 
```bash
sudo rm /etc/nginx/sites-enabled/default && sudo nginx -s reload
```
The welcome page should now be gone.

## Database
This command will take you through a guided wizard to secure MariaDB:
```bash
sudo mysql_secure_installation
```
We mostly use the default values by just pressing Enter. It goes like this:
Enter, 
Enter, 
Enter,
insert a password for the root account, 
repeat the password, 
Enter, 
Enter, 
Enter, 
Enter. 
You should see "Thanks for using MariaDB".

login to MariaDB:
```bash
sudo mysql -u root -p
```
enter the root password you just set.

You should see `MariaDB [(none)]>` on the left of your cursor.
We now create the DB, the DB user and a password. You can adjust the names and password however you like. If you do, remember your changes, since you will need this information later on for the setup. 

Insert this:
```mysql
CREATE SCHEMA `db-ninja-01` DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci;
CREATE USER 'ninja'@'localhost' IDENTIFIED BY 'ninja';
GRANT ALL PRIVILEGES ON `db-ninja-01`.* TO 'ninja'@'localhost';
FLUSH PRIVILEGES;
```        
press enter to run the last line and then exit the db:
```mysql
exit
```
You should see `Bye`.

## NGINX
create config for webpage. This is the example if you run InvoiceNinja behind a NGINX proxy or locally without encryption. If you want to publish InvoiceNinja without a proxy, you would need to get a valid cert with certbot and adjust the server_name.  
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

        location = /index.php {
                include snippets/fastcgi-php.conf;
                fastcgi_pass unix:/run/php/php8.3-fpm.sock;
        }

        location ~ \.php$ {
                return 403;
        }

        location / {
                try_files $uri $uri/ /index.php?$query_string;  
        }

        location ~ /\.ht {
                deny all;
        }

}
```
save and exit.

Check if syntax of your file is ok
```bash
sudo nginx -t
```
## PDF generation
There are multiple options to generate PDFs. One is to use snappdf which chromium to create PDFs locally, the other simpler option is to use phantomjscloud.com. You get 500 PDFs a day for free. Downside is that you have relay on a third party (privacy). Decide which one you want to use. 

### phantomjscloud
For phantomjscloud to work, you need a working an public InvoiceNinja installation. Local only won't work. Go to https://phantomjscloud.com/index.html and create an account. 
In your dashboard, you will see an API Key. 

### SnapPDF
As of today, this current implementation worked in the past, but is for some reason now broken. If you want to use it, you will have to hunt down bugs.  
We need some dependencies. 
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
Right click the link the invoiceninja.tar link and copy the download link

Download the file
```bash
sudo wget https://github.com/invoiceninja/invoiceninja/releases/download/v5.13.1/invoiceninja.tar
```
extract and delete it
```bash
sudo tar -xvf invoiceninja.tar && sudo rm invoiceninja.tar
```

We generate a random key, that we use later:
```bash
openssl rand -base64 32
```

Now we need to edit your .env file.
I added some hashtag comments above the line(s) you need to change.  

copy it and edit the .env file
```bash
sudo nano /usr/share/nginx/invoiceninja/.env
```
insert this:
```bash
APP_NAME="Invoice Ninja"
APP_ENV=production
## insert your app key
APP_KEY=base64:TSMttVrnArwKUlzkHKPYFbNH+pbLBDHdWUWJkp0yTvk=
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
# If you are running a reverse proxy, add the IP here
TRUSTED_PROXIES=192.168.1.10
SCOUT_DRIVER=null

NINJA_ENVIRONMENT=selfhost

# change this your PDF creator
#options - snappdf / phantom / hosted_ninja
PDF_GENERATOR=phantom
# if you go with phantom, insert the API key from the webGUI here
PHANTOMJS_KEY='yourAPIKey'
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

set permissions to the nginx user:
```bash
sudo chown -R www-data:www-data /usr/share/nginx/invoiceninja
```
edit crontab to run Laravel Scheduler every minute:
```bash
sudo -u www-data crontab -e
```
insert this at the end:
```bash
* * * * * cd /usr/share/nginx/invoiceninja/ && php artisan schedule:run >> /dev/null 2>&1
```
save and exit

enable webpage:
```bash
sudo ln -s /etc/nginx/sites-available/invoiceninja.conf /etc/nginx/sites-enabled/
```

Check if the NGINX config is good:
```bash
sudo nginx -t
```
reload nginx:
```bash
sudo nginx -s reload
```

## Decide how you want to run InvoiceNinja
There are multiple different way on how to run InvoiceNinja

If you only use it locally without https, you don't have to do anything. 

If you want to use InvoiceNinja with a valid cert and possibly even from remote, you should go with option A.

If you want to use InvoiceNinja with a valid cert behind a NGINX reverse proxy, you should go with option B.

### Option A: Installing certbot to get a valid cert  
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
this will lead you trough the setup process. If something fails, it is probably because you forgot to set the public dns or your firewall blocks port 80 to certbot.  
Anyway, there are a lot of good tutorials online how to setup certbot.
Certbot automatically changes your NGINX .conf file to listen on port 443.

### Option B: running behind a proxy
On the NGXIN reverse proxy, start by creating an empty config site.
```bash
sudo nano /etc/nginx/sites-available/ninja.mydomain.com.conf
```
```conf
server {
    server_name ninja.mydomain.com;
    listen       80;
    listen      [::]:80;

}
```
save and exit.
Enable the site by running:
```bash
sudo ln -s /etc/nginx/sites-available/ninja.mydomain.com.conf /etc/nginx/sites-enabled/
```

create a cert
```bash
sudo certbot --nginx
```

after that, we add the proxy stuff:
```conf
server {
    server_name ninja.mydomain.com;

    listen 443 ssl; # managed by Certbot
    listen [::]:443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/ninja.mydomain.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/ninja.mydomain.com/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

    # This is important or you will get 413 Server error from the proxy side
    client_max_body_size 100M;

    location / {
                    proxy_set_header        Host $host;
                    proxy_set_header        X-Real-IP $remote_addr;
                    proxy_set_header        X-Forwarded-For proxy_add_x_forwarded_for; 
                    proxy_set_header        X-Forwarded-Proto $scheme;
                    proxy_buffering         off;
                    proxy_request_buffering off;
                    proxy_pass_header       Server;
                    # Enter the IP or Internal DNS name of the server Invoice Ninja is installed on
                    proxy_pass              http://192.168.1.2/;
                    proxy_redirect          off;
                    }

}
server {
    if ($host = ninja.mydomain.com) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


    server_name ninja.mydomain.com;
    listen       80;
    listen      [::]:80;
    return 404; # managed by Certbot


}
```

Test your config
```bash
sudo nginx -t
```
apply your config
```bash
sudo nginx -s reload
```

## Split DNS
For some stuff, like pictures on invoices, InvoiceNinja will make use of URLs like ninja.yourdomain.com/picture.

The problem with that is that ninja.yourdomain.com will be answered by a DNS with the a public IPv4 like 80.80.80.1. But that public IP won't work from your local network. So you have to change that by doing a DNS override. 
The goal of that is that instead of your DNS server answering ninja.yourdomain.com with 80.80.80.1, it should answer it with your local proxy IP 192.168.2.10. Or if you don't use a proxy 192.168.2.2. 

You can achive that with eiter adding a DNS overrride to your DNS server. Since this is not always possible, the alternative is to add an override on the ninja instance and the clients themselves.  

If you don't know how to do that, this paragraph should help:
https://github.com/jameskimmel/Nextcloud_Ubuntu/blob/main/nextcloud_behind_NGINX_proxy.md#split-dns-or-hairpin-nat

## Finish the installation
Visit our https://ninja.yourdomain.com/setup address to setup InvoiceNinja  
If you use it locally without SSL, use http and your IP instead (http://192.168.1.2/setup).    
Test PDF should show success.  

If you did not make any changes to the DB settings, you only need to insert 'ninja' as password and can click on 'Test connection'.

Create a User Account, agree to the terms and click submit.  
Now this could take some time. You should be redirected to the URL you defined earlier.  
Don't leave the page and have some patience. If the page is just gray, try to disable pihole or any other adblockers.  

This is it. You are done and hopefully everything is up and running!

## Optional: optimize the artisan queque
For better performance you can install supervisor. Used to work in the past, but currently has some errors. Use it only if you are willing to hunt down bugs.
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
and change the linke QUEUE_CONNECTION to this
```bash
QUEUE_CONNECTION=database
```
save and exit.
Run
```bash
cd /usr/share/nginx/invoiceninja/
sudo -u www-data php artisan optimize
sudo -u www-data php artisan queue:restart
```
Done :)
