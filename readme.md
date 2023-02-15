# Install Mattermost on Ubuntu - step by step full commands

In this documentation, we will be installing mattermost step by step.

We need to install

- MySQL: for database purpose
- Nginx: for https connections

Let's get started.

**NOTE: This script will only work on Ubuntu or Debian based Linux OS**

## Installtion commands

source: https://docs.mattermost.com/install/install-tar.html

As of I am writing the documentation, the latest available mattermost version is `7.7.1`, please check the latest version here at https://mattermost.com/deploy/ and replace it in the `wget` command

```sh
wget https://releases.mattermost.com/7.7.1/mattermost-7.7.1-linux-amd64.tar.gz
tar -xvzf mattermost*.gz
sudo mv mattermost /opt
sudo mkdir /opt/mattermost/data
sudo useradd --system --user-group mattermost
sudo chown -R mattermost:mattermost /opt/mattermost
sudo chmod -R g+w /opt/mattermost
sudo chmod -R g+w /opt/mattermost
```

### Installing the MySQL 

source: https://www.digitalocean.com/community/tutorials/how-to-install-mysql-on-ubuntu-20-04

```sh
sudo apt update
sudo apt install mysql-server
sudo systemctl start mysql.service
```
### Configuring MySQL

**NOTE: replace `password` with password you like to setup.**

```
sudo mysql

>> ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'password';

exit
```

(optional) This command secure your MySQL installtion

```
sudo mysql_secure_installation

```
## Configuring Mattermost DB

source: https://docs.mattermost.com/install/install-tar.html

```
sudo nano /opt/mattermost/config/config.json
```

In the file find for `DriverName` and `DataSource` and update like to MySQL with point to local

```
"DriverName": "mysql",
"DataSource": "root:password@tcp(localhost:3306)/mattermost?charset=utf8mb4,utf8&writeTimeout=30s"
        
```
### Create DB in MySQL

Login to MySQL using `mysql` command

```
mysql -u root -p

>> create database mattermost;
```

## Test mattermost is working 

```
cd /opt/mattermost
sudo -u mattermost bin/mattermost
```
When the server starts, it shows some log information and the text Server is listening on :8065.

On the browser you can hit <ip>:8065 (if the browser keep's loading for too-long, then you needed to allow 8065 port in Firewall)


After testing press ctrl + c to stop mattermost

## Installing as service and startup

```
sudo nano /lib/systemd/system/mattermost.service
```
and put the following commands

```
[Unit]
Description=Mattermost
After=network.target
After=mysql.service
BindsTo=mysql.service
[Service]
Type=notify
ExecStart=/opt/mattermost/bin/mattermost
TimeoutStartSec=3600
KillMode=mixed
Restart=always
RestartSec=10
WorkingDirectory=/opt/mattermost
User=mattermost
Group=mattermost
LimitNOFILE=49152
[Install]
WantedBy=multi-user.target
```
now close the file and execute the following commands to start mattermost in background and enable in start-up

```
sudo systemctl daemon-reload
sudo systemctl status mattermost.service
sudo systemctl start mattermost.service
sudo systemctl enable mattermost.service
```

# Install Nginx and Configuring SSL

For SSL you required domain or subdomain pointing to your server.

## Installing the Nginx

We will install and Certbot.
Certbot is command utils to configure SSL for our domain

source: https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-20-04

```
sudo apt install nginx
sudo apt install certbot python3-certbot-nginx
```

## Creating reverse proxy 

source https://docs.mattermost.com/install/config-proxy-nginx.html


```
sudo nano /etc/nginx/sites-available/mattermost

```
Add the following lines.

Replace `mattermost.example.com` with your domain / subdomin pointing to your server.


```
server {
  listen 80;
  server_name   mattermost.example.com;
  return 301 https://$server_name$request_uri;
}

```
Save the file and execute

```
sudo ln -s /etc/nginx/sites-available/mattermost /etc/nginx/sites-enabled/mattermost
sudo nginx -t
```

You should get output without any errors, now let's restart the server

```
sudo systemctl restart nginx
```

### Installing the SSL ceritificate

Replace `mattermost.example.com` with your domain / subdomin pointing to your server.

```
sudo certbot --nginx -d mattermost.example.com
```

Once the ceritificate is installed **delete** the config and paste the following config there

```
sudo rm /etc/nginx/sites-available/mattermost
sudo nano /etc/nginx/sites-available/mattermost

```
Replace `mattermost.example.com` with your domain / subdomin pointing to your server.

```
upstream backend {
   server localhost:8065;
   keepalive 32;
}

proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=mattermost_cache:10m max_size=3g inactive=120m use_temp_path=off;

server {
  listen 80;
  server_name   mattermost.example.com;
  return 301 https://$server_name$request_uri;
}

server {
   listen 443 ssl http2;
   server_name    mattermost.example.com;

   http2_push_preload on; # Enable HTTP/2 Server Push

   ssl on;
   ssl_certificate /etc/letsencrypt/live/mattermost.example.com/fullchain.pem;
   ssl_certificate_key /etc/letsencrypt/live/mattermost.example.com/privkey.pem;
   ssl_session_timeout 1d;

   # Enable TLS versions (TLSv1.3 is required upcoming HTTP/3 QUIC).
   ssl_protocols TLSv1.2 TLSv1.3;

   # Enable TLSv1.3's 0-RTT. Use $ssl_early_data when reverse proxying to
   # prevent replay attacks.
   #
   # @see: https://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_early_data
   ssl_early_data on;

   ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384';
   ssl_prefer_server_ciphers on;
   ssl_session_cache shared:SSL:50m;
   # HSTS (ngx_http_headers_module is required) (15768000 seconds = six months)
   add_header Strict-Transport-Security max-age=15768000;
   # OCSP Stapling ---
   # fetch OCSP records from URL in ssl_certificate and cache them
   ssl_stapling on;
   ssl_stapling_verify on;

   add_header X-Early-Data $tls1_3_early_data;

   location ~ /api/v[0-9]+/(users/)?websocket$ {
       proxy_set_header Upgrade $http_upgrade;
       proxy_set_header Connection "upgrade";
       client_max_body_size 50M;
       proxy_set_header Host $http_host;
       proxy_set_header X-Real-IP $remote_addr;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       proxy_set_header X-Forwarded-Proto $scheme;
       proxy_set_header X-Frame-Options SAMEORIGIN;
       proxy_buffers 256 16k;
       proxy_buffer_size 16k;
       client_body_timeout 60;
       send_timeout 300;
       lingering_timeout 5;
       proxy_connect_timeout 90;
       proxy_send_timeout 300;
       proxy_read_timeout 90s;
       proxy_http_version 1.1;
       proxy_pass http://backend;
   }

   location / {
       client_max_body_size 50M;
       proxy_set_header Connection "";
       proxy_set_header Host $http_host;
       proxy_set_header X-Real-IP $remote_addr;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       proxy_set_header X-Forwarded-Proto $scheme;
       proxy_set_header X-Frame-Options SAMEORIGIN;
       proxy_buffers 256 16k;
       proxy_buffer_size 16k;
       proxy_read_timeout 600s;
       proxy_cache mattermost_cache;
       proxy_cache_revalidate on;
       proxy_cache_min_uses 2;
       proxy_cache_use_stale timeout;
       proxy_cache_lock on;
       proxy_http_version 1.1;
       proxy_pass http://backend;
   }
}

# This block is useful for debugging TLS v1.3. Please feel free to remove this
# and use the `$ssl_early_data` variable exposed by NGINX directly should you
# wish to do so.
map $ssl_early_data $tls1_3_early_data {
  "~." $ssl_early_data;
  default "";
}

```


Now test and reload the nginx....

```
sudo nginx -t
```

```
sudo service nginx reload
```

Now visit your domain, your mattermost will be up and running.