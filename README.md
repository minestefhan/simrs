# Dokumentasi Instalasi Lengkap (Ubuntu 22.04)

IP server: **192.168.10.2**\
User non-root untuk PM2: **user-project**\
User Nginx + PHP-FPM: **user-project**\
Lokasi project frontend/socket/kiosk/viewer/ereservasi:
**/opt/project/**\
Lokasi backend Laravel: **/service**\
Port aplikasi: - Frontend Typescript JS: **2222** - Socket.io:
**2223** - Kiosk Angular: **2224** - Viewer: **2225** - E-Reservasi:
**2226** - Backend: **PHP-FPM 7.4 (Laravel)**

------------------------------------------------------------------------

# A) BASE INSTALL

## 1) Update repositori

``` bash
sudo apt update && sudo apt -y upgrade
```

------------------------------------------------------------------------

## 2) Instal Nginx (mainline repo)

``` bash
sudo nano /etc/apt/sources.list.d/nginx.list
```

Tambahkan:

    deb [arch=amd64] http://nginx.org/packages/mainline/ubuntu/ bionic nginx
    deb-src http://nginx.org/packages/mainline/ubuntu/ bionic nginx

Import key:

``` bash
wget http://nginx.org/keys/nginx_signing.key
sudo apt-key add nginx_signing.key
sudo apt update
```

Hapus versi lama lalu install:

``` bash
sudo apt remove nginx nginx-common nginx-full nginx-core
sudo apt install nginx
```

------------------------------------------------------------------------

## 3) Ubah user Nginx menjadi `user-project`

``` bash
sudo nano /etc/nginx/nginx.conf
```

Ubah:

    user user-project;

Restart:

``` bash
sudo systemctl restart nginx
```

------------------------------------------------------------------------

## 4) Instal PHP-FPM 7.4 + ekstensi lengkap

``` bash
sudo apt install -y software-properties-common
sudo add-apt-repository ppa:ondrej/php -y
sudo apt update
```

``` bash
sudo apt install -y \
  php7.4-fpm php7.4-cli php7.4-common \
  php7.4-mbstring php7.4-xml php7.4-curl php7.4-zip \
  php7.4-gd php7.4-bcmath php7.4-intl php7.4-soap \
  php7.4-opcache php7.4-readline php7.4-redis \
  php7.4-pgsql php7.4-json php7.4-pdo php7.4-mysql \
  php7.4-dom
```

------------------------------------------------------------------------

## 5) Konfigurasi PHP-FPM `/etc/php/7.4/fpm/pool.d/www.conf`

``` bash
sudo nano /etc/php/7.4/fpm/pool.d/www.conf
```

Set:

    user = user-project
    group = user-project

    listen = /run/php/php7.4-fpm.sock
    listen.owner = user-project
    listen.group = user-project
    listen.mode = 06606

    pm = ondemand
    pm.max_children = 256
    pm.process_idle_timeout = 10s
    pm.max_requests = 1000

    pm.status_path = /status

Restart:

``` bash
sudo systemctl restart php7.4-fpm
```

------------------------------------------------------------------------

## 6) Setting `php.ini` (postgres + upload)

``` bash
sudo nano /etc/php/7.4/fpm/php.ini
```

Pastikan:

    extension=pdo_pgsql
    extension=pgsql

Ubah:

    post_max_size = 110M
    upload_max_filesize = 100M
    max_execution_time = 100

Restart:

``` bash
sudo systemctl restart php7.4-fpm
```

------------------------------------------------------------------------

## 7) Install driver MongoDB untuk PHP 7.4 (SKIP)

> **Dihapus sesuai permintaan** --- tidak ada instalasi MongoDB maupun
> driver MongoDB.

------------------------------------------------------------------------

## 8) Disable Apache

``` bash
sudo service apache2 stop
sudo update-rc.d apache2 disable
```

------------------------------------------------------------------------

## 9) Instal PostgreSQL 14

``` bash
sudo apt install -y wget ca-certificates
wget -qO - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo tee /etc/apt/trusted.gpg.d/postgresql.asc
echo "deb http://apt.postgresql.org/pub/repos/apt jammy-pgdg main" | sudo tee /etc/apt/sources.list.d/pgdg.list
sudo apt update
sudo apt install -y postgresql-14 postgresql-client-14
```

------------------------------------------------------------------------

## 10) Konfigurasi PostgreSQL (port, listen, pg_hba)

Edit:

``` bash
sudo nano /etc/postgresql/14/main/postgresql.conf
```

Set:

    listen_addresses = '*'
    port = 5433
    effective_cache_size = 16GB
    shared_buffers = 4GB
    work_mem = 16MB
    maintenance_work_mem = 128MB
    wal_level = replica
    max_wal_size = 1GB
    min_wal_size = 80MB
    max_connections = 500

Edit pg_hba:

``` bash
sudo nano /etc/postgresql/14/main/pg_hba.conf
```

Tambahkan:

    host    all             all             0.0.0.0/0               md5
    host    all             all             ::/0                    md5

Restart:

``` bash
sudo systemctl restart postgresql
```

------------------------------------------------------------------------

## 11) Ganti password user postgres

``` bash
sudo -u postgres psql
```

``` sql
ALTER USER postgres WITH PASSWORD 'PASSWORD_BARU';
\q
```

------------------------------------------------------------------------

## 12) Instal Git + Composer

``` bash
sudo apt install -y git composer
git config --global credential.helper store
```

------------------------------------------------------------------------

## 13) Install NVM (user-project, tanpa sudo)

``` bash
su - user-project
```

``` bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
```

------------------------------------------------------------------------

## 14) Install Node.js 14 & 16

``` bash
nvm install 14
nvm install 16
nvm alias default 16
```

------------------------------------------------------------------------

## 15) Install PM2, Yarn, Angular CLI

``` bash
npm install -g pm2
npm install -g yarn
npm install -g @angular/cli
```

------------------------------------------------------------------------

# B) NGINX CONFIG

## 1) nginx.conf stabil

``` bash
sudo nano /etc/nginx/nginx.conf
```

\`\`\`nginx name=/etc/nginx/nginx.conf user user-project;
worker_processes auto; pid /run/nginx.pid; include
/etc/nginx/modules-enabled/\*.conf;

events { worker_connections 4096; multi_accept on; }

http { sendfile on; tcp_nopush on; tcp_nodelay on; keepalive_timeout 65;
types_hash_max_size 4096;

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    server_tokens off;

    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

    include /etc/nginx/conf.d/*.conf;

}


    ---

    ## 2) default.conf gateway (backend + proxy)
    ```bash
    sudo nano /etc/nginx/conf.d/default.conf

``` nginx
server {
        listen 80;
        listen 443 reuseport ssl http2 backlog=1024 rcvbuf=64000 sndbuf=200000 fastopen=500;

        server_name simrs-grhasia.jogjaprov.go.id;

        add_header Access-Control-Allow-Origin '*' always;
        add_header access-control-allow-origin '*' always;
        add_header Access-Control-Allow-Methods 'OPTIONS, GET, POST, PUT, DELETE' always;
        add_header access-control-allow-methods 'OPTIONS, GET, POST, PUT, DELETE' always;
        add_header Referrer-Policy 'no-referrer' always;

        proxy_hide_header X-Powered-By;

        location = /favicon.ico { access_log off; log_not_found off; }
        location = /robots.txt  { access_log off; log_not_found off; }

        index index.html index.php;

        ssl_certificate /etc/nginx/ssl/jogjaprov_go_id.crt;
        ssl_certificate_key /etc/nginx/ssl/jogjaprov_go_id.key;
        ssl_trusted_certificate /etc/nginx/ssl/jogjaprov_go_id.crt;

        location /socket.io/ {
                limit_req zone=sock burst=5;
                limit_req_status 444;

                proxy_http_version 1.1;
                proxy_hide_header 'Access-Control-Allow-Origin';
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection 'upgrade';
                proxy_pass "http://127.0.0.1:8881";
        }

        location /kiosk/ {
           proxy_pass http://127.0.0.1:2224/;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        location /viewer/ {
           proxy_pass http://127.0.0.1:4300/;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        location /e-reservasi/ {
           proxy_pass http://127.0.0.1:4200/;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        location ^~ /service {
                alias /opt/rsj_grhasia/backend/public;
                try_files $uri $uri/ @laravelService;

                location ~ \.php {
                        include fastcgi_params;
                        fastcgi_param SCRIPT_FILENAME $request_filename;
                        fastcgi_pass 127.0.0.1:9000;
                }
        }

        location @laravelService {
                rewrite /service/(.*)$ /service/index.php/$1 last;
        }

        location ^~ /eis {
                alias /opt/rsj_grhasia/eis/public;
                try_files $uri $uri/ @laravelEIS;

                location ~ \.php {
                        include fastcgi_params;
                        fastcgi_param SCRIPT_FILENAME $request_filename;
                        fastcgi_pass 127.0.0.1:9000;
                }
        }

        location @laravelEIS {
                rewrite /eis/(.*)$ /eis/index.php?/$1 last;
        }

        location / {
                root /opt/rsj_grhasia/frontend;
                try_files $uri @express;
        }

        location @express {
                proxy_pass http://127.0.0.1:2222;
                proxy_set_header Host $http_host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_http_version 1.1;
        }
}

server {
    listen 80;
    server_name simrs-grhasia.jogjaprov.go.id;

    location / {
        return 303 https://$host$request_uri;
    }

    location ^~ /.well-known/acme-challenge/ {
        default_type "text/plain";
        root /var/www/letsencrypt;
    }

    location ^~ /.well-known/pki-validation/ {
        default_type "text/plain";
        root /var/www/letsencrypt;
    }
}
```

## 3) Copy SSL placeholder

``` bash
cd /opt
sudo mkdir ssl
cd /opt/ssl
sudo nano default_blank.crt
sudo nano default_blank.key
```

------------------------------------------------------------------------

## 4) Test & restart Nginx

``` bash
sudo nginx -t
sudo service nginx restart
```

------------------------------------------------------------------------

# C) MENJALANKAN APLIKASI MENGGUNAKAN PM2

Pastikan seluruh source code sudah berada di `/opt/rsj_grhasia/`.

---

## 1. Frontend SIMRS (Port 2222)

```bash
cd /opt/rsj_grhasia/frontend

npm install

pm2 start npm --name simrs -- run run-app
```

---

## 2. Socket.IO (Port 8881)

```bash
cd /opt/rsj_grhasia/socket

npm install

pm2 start "npm start -- --port 8881" \
    --name socket
```

Apabila menggunakan file server langsung:

```bash
pm2 start server.js \
    --name socket \
    -- --port 8881
```

---

## 3. Kiosk (Port 2224)

```bash
cd /opt/rsj_grhasia/kiosk

npm install

pm2 start "ng serve --host 0.0.0.0 --port 2224" \
    --name kiosk
```

---

## 4. Viewer (Port 4300)

```bash
cd /opt/rsj_grhasia/viewer

npm install

pm2 start "ng serve --host 0.0.0.0 --port 4300" \
    --name viewer
```

---

## 5. E-Reservasi (Port 4200)

```bash
cd /opt/rsj_grhasia/e-reservasi

npm install

pm2 start "ng serve --host 0.0.0.0 --port 4200" \
    --name e-reservasi
```

---

## 6. Melihat Status Service

```bash
pm2 list
```

---

## 7. Monitoring Log

```bash
pm2 logs
```

atau

```bash
pm2 logs simrs
pm2 logs socket
pm2 logs kiosk
pm2 logs viewer
pm2 logs e-reservasi
```

---

## 8. Restart Service

```bash
pm2 restart simrs
pm2 restart socket
pm2 restart kiosk
pm2 restart viewer
pm2 restart e-reservasi
```

Restart seluruh service

```bash
pm2 restart all
```

---

## 9. Stop Service

```bash
pm2 stop simrs
pm2 stop socket
pm2 stop kiosk
pm2 stop viewer
pm2 stop e-reservasi
```

---

## 10. Hapus Service

```bash
pm2 delete simrs
pm2 delete socket
pm2 delete kiosk
pm2 delete viewer
pm2 delete e-reservasi
```

---

## 11. Simpan Konfigurasi PM2

```bash
pm2 save
```

---

## 12. Auto Start Setelah Reboot

```bash
pm2 startup systemd
```

Jalankan command yang dihasilkan PM2, kemudian:

```bash
pm2 save
```

Verifikasi:

```bash
systemctl status pm2-user-project
```

------------------------------------------------------------------------

# D) REDIS (Backend Laravel)

``` bash
composer require predis/predis
sudo apt install redis-server
sudo nano /etc/redis/redis.conf
```

Ubah:

    supervised systemd
    requirepass lCa34x2RmtXFgTSa8SOvwLTTFzQVI6ffN0hlClhOfTTZPW+u8tF1IBdOXqhkIT5NoxBmef9Umfxm
    protected-mode no
    # bind 127.0.0.1 ::1

Restart:

``` bash
sudo systemctl restart redis
sudo systemctl status redis
```

Tambahkan di `.env`:

    CACHE_DRIVER=redis
    SESSION_DRIVER=redis
    QUEUE_DRIVER=sync
    CONNECTION_SESSION=session

    REDIS_HOST=127.0.0.1
    REDIS_PASSWORD=null
    REDIS_PORT=6379

Tambahkan di `config/database.php`:

    'session' => [
        'host' => env('REDIS_HOST', '127.0.0.1'),
        'password' => env('REDIS_PASSWORD', null),
        'port' => env('REDIS_PORT', 6379),
        'database' => 1,
    ],

`config/session.php`:

    'connection' => ENV('CONNECTION_SESSION', null),

``` bash
php artisan cache:clear
php artisan config:cache
```

------------------------------------------------------------------------

# E) INSTALASI SSL LET'S ENCRYPT (CERTBOT)

## 1. Install Certbot

```bash
sudo apt update

sudo apt install -y certbot python3-certbot-nginx
```

Verifikasi:

```bash
certbot --version
```

---

## 2. Pastikan DNS Sudah Mengarah ke Server

Cek:

```bash
nslookup simrs-grhasia.jogjaprov.go.id
```

atau

```bash
dig simrs-grhasia.jogjaprov.go.id
```

Pastikan IP yang tampil adalah IP Public server.

---

## 3. Pastikan Port Firewall Terbuka

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

Jika menggunakan firewalld

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

---

## 4. Pastikan Nginx Berjalan

```bash
sudo nginx -t
sudo systemctl restart nginx
sudo systemctl status nginx
```

---

## 5. Generate SSL

```bash
sudo certbot certonly \
    --nginx \
    -d simrs-grhasia.jogjaprov.go.id
```

atau jika ingin langsung redirect HTTP ke HTTPS

```bash
sudo certbot --nginx \
    -d simrs-grhasia.jogjaprov.go.id
```

---

## 6. Lokasi Sertifikat

Certbot akan membuat sertifikat pada:

```text
/etc/letsencrypt/live/simrs-grhasia.jogjaprov.go.id/
```

Berisi:

```text
fullchain.pem
privkey.pem
chain.pem
cert.pem
```

---

## 7. Konfigurasi SSL pada Nginx

Edit:

```bash
sudo nano /etc/nginx/conf.d/default.conf
```

Gunakan:

```nginx
ssl_certificate /etc/letsencrypt/live/simrs-grhasia.jogjaprov.go.id/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/simrs-grhasia.jogjaprov.go.id/privkey.pem;
ssl_trusted_certificate /etc/letsencrypt/live/simrs-grhasia.jogjaprov.go.id/chain.pem;
```

---

## 8. Test Konfigurasi

```bash
sudo nginx -t
```

Jika berhasil:

```bash
sudo systemctl reload nginx
```

---

## 9. Test SSL

```bash
curl -Iv https://simrs-grhasia.jogjaprov.go.id
```

atau

```bash
openssl s_client -connect simrs-grhasia.jogjaprov.go.id:443
```

---

## 10. Auto Renewal

Cek timer:

```bash
systemctl list-timers | grep certbot
```

Test renewal:

```bash
sudo certbot renew --dry-run
```

Renew manual:

```bash
sudo certbot renew
```

---

## 11. Reload Nginx Setelah Renewal

Buat hook:

```bash
sudo mkdir -p /etc/letsencrypt/renewal-hooks/deploy
```

```bash
sudo nano /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
```

Isi:

```bash
#!/bin/bash

systemctl reload nginx
```

Permission:

```bash
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
```

---

## 12. Verifikasi Sertifikat

```bash
sudo certbot certificates
```

Contoh output:

```text
Certificate Name: simrs-grhasia.jogjaprov.go.id
Domains: simrs-grhasia.jogjaprov.go.id
Expiry Date: 2026-10-15
Certificate Path:
/etc/letsencrypt/live/simrs-grhasia.jogjaprov.go.id/fullchain.pem
Private Key Path:
/etc/letsencrypt/live/simrs-grhasia.jogjaprov.go.id/privkey.pem
```
------------------------------------------------------------------------



------------------------------------------------------------------------

# E) SSL LET'S ENCRYPT (FREE)

Referensi:\
https://www.nginx.com/blog/using-free-ssltls-certificates-from-lets-encrypt-with-nginx/

------------------------------------------------------------------------

# F) GETSSL (SSL Gratis Alternatif)

``` bash
curl --silent https://raw.githubusercontent.com/srvrco/getssl/latest/getssl > getssl ; chmod 700 getssl
mkdir -p /var/www/letsencrypt/.well-known/acme-challenge
cd /opt/cert
./getssl -c [nama_domain]
```

Config:

``` bash
nano /root/.getssl/[nama_domain]/getssl.cfg
```

    CA="https://acme-staging-v02.api.letsencrypt.org"
    ACL=('/var/www/.well-known/acme-challenge')
    USE_SINGLE_ACL="true"
    DOMAIN_CERT_LOCATION="/opt/cert/digicare.crt"
    DOMAIN_KEY_LOCATION="/opt/cert/digicare.key"
    DOMAIN_PEM_LOCATION="/opt/cert/digicare.pem"
    RELOAD_CMD="systemctl reload nginx"
    ACCOUNT_EMAIL="ramdanegie@gmail.com"
    ACCOUNT_KEY_LENGTH=2048
    AGREEMENT="https://letsencrypt.org/documents/LE-SA-v1.2-November-15-2017.pdf"

``` bash
nano /etc/hosts
# tambahkan
127.0.0.1 [nama_domain]

./getssl [nama_domain]
```

Jika sukses, ubah CA ke live:

    CA="https://acme-v02.api.letsencrypt.org"

``` bash
./getssl -f [nama_domain]
```

Restart Nginx.

------------------------------------------------------------------------

# G) BACKUP & RESTORE POSTGRES

Backup:

``` bash
pg_dump -v -Fc -Z 9 -f namafile.backup nama_db
```

Restore:

``` bash
pg_restore -v -d nama_db filedatabase.backup
```

Hapus DB:

``` bash
dropdb nama_db
```

Buat DB:

``` bash
createdb nama_db
```

------------------------------------------------------------------------

# H) BACKUP OTOMATIS CRON

``` bash
sudo su
cd /var/lib/postgresql
mkdir backups-transmedic-db
cd backups-transmedic-db
nano backup.sh
```

Isi:

    pg_dump -U postgres  -v -Fc -Z 9 -f  /var/lib/postgresql/backups-transmedic-db/backup_auto_`date +\%Y\%m\%d`.backup rsud_bagaswaras 
    pg_dump -U postgres  -v -Fc -Z 9 -f  /var/lib/postgresql/backups-transmedic-db/backup_auto_`date +\%Y\%m\%d`.backup rs_advent --verbose 2>backup.log

Delete old:

``` bash
nano delete_2_month_ago.sh
```

Isi:

    rm -rf backup_auto_`date -d $(date -d "7 day ago" +%Y%m%d) +%Y%m%d`*

Cron:

``` bash
su - postgres
crontab -e
```

Tambahkan:

    0 01 * * * sh /var/lib/postgresql/backups-transmedic-db/backup.sh
    0 01 * * * sh /var/lib/postgresql/backups-transmedic-db/delete_2_month_ago.sh

------------------------------------------------------------------------

# I) CHOKIDAR FIX

``` bash
echo fs.inotify.max_user_watches=524288 | sudo tee -a /etc/sysctl.conf && sudo sysctl -p
sudo fs.inotify.max_user_watches=524288
sudo sysctl --system
```

------------------------------------------------------------------------

# J) ALIAS

``` bash
sudo nano ~/.bash_aliases
```

Isi:

    alias kahareup='cd /opt/rs'
    alias pull='sudo git pull'
    alias develop='cd /opt/dev/rs_amc'

``` bash
source ~/.bashrc
```

------------------------------------------------------------------------

# K) OPENVPN

``` bash
sudo apt install openvpn
sudo openvpn --client --config jasmed.ovpn &
sudo openvpn jasmed.ovpn
```

------------------------------------------------------------------------

# L) TIPS LAINNYA

-   Cek reboot:

``` bash
last | grep reboot
```

-   Cari string di folder:

``` bash
grep -rnw foldernama -e 'patternname'
```
