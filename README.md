# Ubuntu-22.04

1. [User](#user)
1. [Nginx + Brotli](#nginx)
1. [Docker](#docker)
1. [App](#app)

## User

```
ssh root@happ.od.ua
```

```
apt update -y
```

```
adduser happ
```

```
usermod -aG sudo happ
```

```
rsync --archive --chown=happ:happ ~/.ssh /home/happ
```

```
nano /etc/ssh/sshd_config
```

```
PermitRootLogin no
```

```
service sshd restart
```

```
ssh happ@happ.od.ua
```

## Nginx

> Nginx, Brotli, SSL, HTTP2

```
sudo apt install dpkg-dev build-essential gnupg2 git gcc cmake libpcre3 libpcre3-dev zlib1g zlib1g-dev openssl libssl-dev curl unzip -y
```
```
curl -L https://nginx.org/keys/nginx_signing.key | sudo apt-key add -
```
```
sudo nano /etc/apt/sources.list.d/nginx.list
```

```
deb http://nginx.org/packages/ubuntu/ focal nginx
deb-src http://nginx.org/packages/ubuntu/ focal nginx
```

```
sudo apt update -y
```

```
cd /usr/local/src
```

```
sudo apt source nginx
```

```
sudo apt build-dep nginx -y
```

```
sudo git clone --recursive https://github.com/google/ngx_brotli.git
```

```
cd /usr/local/src/nginx-*/
```

```
sudo nano debian/rules
```

```
--add-module=/usr/local/src/ngx_brotli
```

```
sudo dpkg-buildpackage -b -uc -us
```

```
sudo dpkg -i /usr/local/src/*.deb
```

```
sudo nano /etc/nginx/conf.d/happ.od.ua.conf
```

```
server {
        listen 80;
        listen [::]:80;

        server_name happ.od.ua www.happ.od.ua;

        location / {
            try_files $uri $uri/ =404;
        }
}
```

```
sudo nginx -t
```

```
sudo systemctl restart nginx
```

```
sudo snap install core
```

```
sudo snap refresh core
```

```
sudo snap install --classic certbot
```

```
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```

```
sudo certbot --nginx -d happ.od.ua -d www.happ.od.ua
```

```
sudo nano /etc/nginx/conf.d/happ.od.ua.conf
```

```
server {
    server_name happ.od.ua www.happ.od.ua;
    
    listen [::]:443 http2 ssl ipv6only=on; # managed by Certbot
    listen 443 http2 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/happ.od.ua/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/happ.od.ua/privkey.pem; # managed by Certbot
    #include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

    ssl_protocols      TLSv1.3 TLSv1.2;
    ssl_ciphers EECDH+CHACHA20:EECDH+AES128:RSA+AES128:EECDH+AES256:RSA+AES256:EECDH+3DES:RSA+3DES:!MD5;
    ssl_prefer_server_ciphers on;
    ssl_session_cache    shared:SSL:20m;
    ssl_session_timeout  20m;

    add_header Strict-Transport-Security "max-age=15768000; includeSubDomains" always;

    access_log /var/log/nginx/happ.od.ua.access.log;
    error_log /var/log/nginx/happ.od.ua.error.log;

    root /var/www/happ.od.ua/htdocs/;
    index index.php;
    charset UTF-8;

    location / {
#       add_header 'Access-Control-Allow-Origin' "$http_origin";
#       add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS, DELETE, PUT';
#       add_header 'Access-Control-Allow-Credentials' 'true';
#       add_header 'Access-Control-Allow-Headers' 'User-Agent,Keep-Alive,Content-Type';

#       proxy_set_header   X-Forwarded-For $remote_addr;
#       proxy_set_header   Host $http_host;
        proxy_pass http://127.0.0.1:8080;
        client_max_body_size 32m;
        proxy_redirect off;
        proxy_read_timeout 300;
        proxy_connect_timeout 300;
        proxy_send_timeout 300;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_headers_hash_max_size 512;
        proxy_headers_hash_bucket_size 128;
    }

    location ^~ /.well-known/acme-challenge/ {
        default_type "text/plain";
        root /var/www/happ.od.ua/htdocs/;
    }
    
    location = /.well-known/acme-challenge/ {
        return 404;
    }

    location = /robots.txt {
        allow all;
        log_not_found off;
        access_log off;
    }

    location ~ /\.svn/* {
        deny all;
    }

    location ~ /\.git/* {
        deny all;
    }

    # Deny all attempts to access hidden files such as .htaccess, .htpasswd, .DS_Store (Mac).
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }
    
    # Basic
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    types_hash_max_size 2048;
    server_tokens off;
    ignore_invalid_headers on;

    # Decrease default timeouts to drop slow clients
    keepalive_timeout 40s;
    send_timeout 20s;
    client_header_timeout 20s;
    client_body_timeout 20s;
    reset_timedout_connection on;
        
    ## Improves TTFB by using a smaller SSL buffer than the nginx default
    ssl_buffer_size 8k;

    ## Enables OCSP stapling
    ssl_stapling on;
    resolver 8.8.8.8 8.8.4.4;
    ssl_stapling_verify on;

    ## Send header to tell the browser to prefer https to http traffic
    add_header Strict-Transport-Security max-age=31536000 always;

    # TTFB optimization
    #tcp_nodelay on;
    #tcp_nopush on;
    #sendfile on;

    #file uploads
    client_max_body_size 128m;

    # Gzip
    gzip on;
    gzip_disable "msie6";
    gzip_vary off;
    gzip_proxied any;
    gzip_comp_level 5;
    gzip_min_length 1000;
    gzip_buffers 16 8k;
    gzip_http_version 1.1;
    gzip_types
        application/atom+xml
        application/javascript
        application/json
        application/ld+json
        application/manifest+json
        application/rss+xml
        application/vnd.geo+json
        application/vnd.ms-fontobject
        application/x-font-ttf
        application/x-web-app-manifest+json
        application/xhtml+xml
        application/xml
        font/opentype
        image/bmp
        image/svg+xml
        image/x-icon
        text/cache-manifest
        text/css
        text/plain
        text/vcard
        text/vnd.rim.location.xloc
        text/vtt
        text/x-component
        text/x-cross-domain-policy;

    # Brotli
    brotli on;
    brotli_comp_level 6;
    brotli_types
        text/xml
        image/svg+xml
        application/x-font-ttf
        image/vnd.microsoft.icon
        application/x-font-opentype
        application/json
        font/eot
        application/vnd.ms-fontobject
        application/javascript
        font/otf
        application/xml
        application/xhtml+xml
        text/javascript
        application/x-javascript
        text/$;

}

server {
    if ($host = www.happ.od.ua) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


    if ($host = happ.od.ua) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


    listen 80;
    listen [::]:80;

    server_name happ.od.ua www.happ.od.ua;
    return 404; # managed by Certbot

}
```

# Docker
> Docker, Docker Compose

```
sudo apt update
```

```
sudo apt install apt-transport-https ca-certificates curl software-properties-common
```

```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

```
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

```
sudo apt update 
```

```
sudo apt-cache policy docker-ce
```

```
sudo apt install docker-ce
```

```
sudo usermod -aG docker ${USER}
```

```
su - ${USER}
```

```
sudo usermod -aG docker happ
```

```
mkdir -p ~/.docker/cli-plugins/
```

```
curl -SL https://github.com/docker/compose/releases/download/v2.3.3/docker-compose-linux-x86_64 -o ~/.docker/cli-plugins/docker-compose
```

```
chmod +x ~/.docker/cli-plugins/docker-compose
```

# App

```
cd ~/.ssh/
```

```
ssh-keygen
```

```
cd /home/happ
```

```
git clone git@github.com:happ-admin/happ.git
```

```
cd happ
```

```
docker compose up -d
```

# Autostart app on reload

```
sudo nano /etc/systemd/system/happ.service
```

```
[Unit]
Description=Happ Service
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/home/happ/happ
ExecStart=/home/happ/.docker/cli-plugins/docker-compose up -d
ExecStop=/home/happ/.docker/cli-plugins/docker-compose stop
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
```

```
sudo systemctl enable happ
```

```
sudo systemctl daemon-reload
```

```
sudo systemctl restart happ.service
```

```
sudo journalctl -u happ.service
```

# Reload Server

```
sudo reboot now
```
