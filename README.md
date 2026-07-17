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
   limit_req_zone $binary_remote_addr zone=sock:10m rate=10r/s;
    

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

# M) www.conf

``` bash
; Start a new pool named 'www'.
; the variable $pool can be used in any directive and will be replaced by the
; pool name ('www' here)
[www]

; Per pool prefix
; It only applies on the following directives:
; - 'access.log'
; - 'slowlog'
; - 'listen' (unixsocket)
; - 'chroot'
; - 'chdir'
; - 'php_values'
; - 'php_admin_values'
; When not set, the global prefix (or /usr) applies instead.
; Note: This directive can also be relative to the global prefix.
; Default Value: none
;prefix = /path/to/pools/$pool

; Unix user/group of processes
; Note: The user is mandatory. If the group is not set, the default user's group
;       will be used.
user = www-data
group = www-data

; The address on which to accept FastCGI requests.
; Valid syntaxes are:
;   'ip.add.re.ss:port'    - to listen on a TCP socket to a specific IPv4 address on
;                            a specific port;
;   '[ip:6:addr:ess]:port' - to listen on a TCP socket to a specific IPv6 address on
;                            a specific port;
;   'port'                 - to listen on a TCP socket to all addresses
;                            (IPv6 and IPv4-mapped) on a specific port;
;   '/path/to/unix/socket' - to listen on a unix socket.
; Note: This value is mandatory.
;listen = /run/php/php7.4-fpm.sock
listen = 127.0.0.1:9000

; Set listen(2) backlog.
; Default Value: 511 (-1 on FreeBSD and OpenBSD)
;listen.backlog = 511

; Set permissions for unix socket, if one is used. In Linux, read/write
; permissions must be set in order to allow connections from a web server. Many
; BSD-derived systems allow connections regardless of permissions. The owner
; and group can be specified either by name or by their numeric IDs.
; Default Values: user and group are set as the running user
;                 mode is set to 0660
listen.owner = www-data
listen.group = www-data
;listen.mode = 0660
; When POSIX Access Control Lists are supported you can set them using
; these options, value is a comma separated list of user/group names.
; When set, listen.owner and listen.group are ignored
;listen.acl_users =
;listen.acl_groups =

; List of addresses (IPv4/IPv6) of FastCGI clients which are allowed to connect.
; Equivalent to the FCGI_WEB_SERVER_ADDRS environment variable in the original
; PHP FCGI (5.2.2+). Makes sense only with a tcp listening socket. Each address
; must be separated by a comma. If this value is left blank, connections will be
; accepted from any ip address.
; Default Value: any
;listen.allowed_clients = 127.0.0.1

; Specify the nice(2) priority to apply to the pool processes (only if set)
; The value can vary from -19 (highest priority) to 20 (lower priority)
; Note: - It will only work if the FPM master process is launched as root
;       - The pool processes will inherit the master process priority
;         unless it specified otherwise
; Default Value: no set
; process.priority = -19

; Set the process dumpable flag (PR_SET_DUMPABLE prctl) even if the process user
; or group is differrent than the master process user. It allows to create process
; core dump and ptrace the process for the pool user.
; Default Value: no
; process.dumpable = yes

; Choose how the process manager will control the number of child processes.
; Possible Values:
;   static  - a fixed number (pm.max_children) of child processes;
;   dynamic - the number of child processes are set dynamically based on the
;             following directives. With this process management, there will be
;             always at least 1 children.
;             pm.max_children      - the maximum number of children that can
;                                    be alive at the same time.
;             pm.start_servers     - the number of children created on startup.
;             pm.min_spare_servers - the minimum number of children in 'idle'
;                                    state (waiting to process). If the number
;                                    of 'idle' processes is less than this
;                                    number then some children will be created.
;             pm.max_spare_servers - the maximum number of children in 'idle'
;                                    state (waiting to process). If the number
;                                    of 'idle' processes is greater than this
;                                    number then some children will be killed.
;  ondemand - no children are created at startup. Children will be forked when
;             new requests will connect. The following parameter are used:
;             pm.max_children           - the maximum number of children that
;                                         can be alive at the same time.
;             pm.process_idle_timeout   - The number of seconds after which
;                                         an idle process will be killed.
; Note: This value is mandatory.
pm = dynamic

; The number of child processes to be created when pm is set to 'static' and the
; maximum number of child processes when pm is set to 'dynamic' or 'ondemand'.
; This value sets the limit on the number of simultaneous requests that will be
; served. Equivalent to the ApacheMaxClients directive with mpm_prefork.
; Equivalent to the PHP_FCGI_CHILDREN environment variable in the original PHP
; CGI. The below defaults are based on a server without much resources. Don't
; forget to tweak pm.* to fit your needs.
; Note: Used when pm is set to 'static', 'dynamic' or 'ondemand'
; Note: This value is mandatory.
pm.max_children = 5

; The number of child processes created on startup.
; Note: Used only when pm is set to 'dynamic'
; Default Value: (min_spare_servers + max_spare_servers) / 2
pm.start_servers = 2

; The desired minimum number of idle server processes.
; Note: Used only when pm is set to 'dynamic'
; Note: Mandatory when pm is set to 'dynamic'
pm.min_spare_servers = 1

; The desired maximum number of idle server processes.
; Note: Used only when pm is set to 'dynamic'
; Note: Mandatory when pm is set to 'dynamic'
pm.max_spare_servers = 3

; The number of seconds after which an idle process will be killed.
; Note: Used only when pm is set to 'ondemand'
; Default Value: 10s
;pm.process_idle_timeout = 10s;

; The number of requests each child process should execute before respawning.
; This can be useful to work around memory leaks in 3rd party libraries. For
; endless request processing specify '0'. Equivalent to PHP_FCGI_MAX_REQUESTS.
; Default Value: 0
;pm.max_requests = 500

; The URI to view the FPM status page. If this value is not set, no URI will be
; recognized as a status page. It shows the following informations:
;   pool                 - the name of the pool;
;   process manager      - static, dynamic or ondemand;
;   start time           - the date and time FPM has started;
;   start since          - number of seconds since FPM has started;
;   accepted conn        - the number of request accepted by the pool;
;   listen queue         - the number of request in the queue of pending
;                          connections (see backlog in listen(2));
;   max listen queue     - the maximum number of requests in the queue
;                          of pending connections since FPM has started;
;   listen queue len     - the size of the socket queue of pending connections;
;   idle processes       - the number of idle processes;
;   active processes     - the number of active processes;
;   total processes      - the number of idle + active processes;
;   max active processes - the maximum number of active processes since FPM
;                          has started;
;   max children reached - number of times, the process limit has been reached,
;                          when pm tries to start more children (works only for
;                          pm 'dynamic' and 'ondemand');
; Value are updated in real time.
; Example output:
;   pool:                 www
;   process manager:      static
;   start time:           01/Jul/2011:17:53:49 +0200
;   start since:          62636
;   accepted conn:        190460
;   listen queue:         0
;   max listen queue:     1
;   listen queue len:     42
;   idle processes:       4
;   active processes:     11
;   total processes:      15
;   max active processes: 12
;   max children reached: 0
;
; By default the status page output is formatted as text/plain. Passing either
; 'html', 'xml' or 'json' in the query string will return the corresponding
; output syntax. Example:
;   http://www.foo.bar/status
;   http://www.foo.bar/status?json
;   http://www.foo.bar/status?html
;   http://www.foo.bar/status?xml
;
; By default the status page only outputs short status. Passing 'full' in the
; query string will also return status for each pool process.
; Example:
;   http://www.foo.bar/status?full
;   http://www.foo.bar/status?json&full
;   http://www.foo.bar/status?html&full
;   http://www.foo.bar/status?xml&full
; The Full status returns for each process:
;   pid                  - the PID of the process;
;   state                - the state of the process (Idle, Running, ...);
;   start time           - the date and time the process has started;
;   start since          - the number of seconds since the process has started;
;   requests             - the number of requests the process has served;
;   request duration     - the duration in µs of the requests;
;   request method       - the request method (GET, POST, ...);
;   request URI          - the request URI with the query string;
;   content length       - the content length of the request (only with POST);
;   user                 - the user (PHP_AUTH_USER) (or '-' if not set);
;   script               - the main script called (or '-' if not set);
;   last request cpu     - the %cpu the last request consumed
;                          it's always 0 if the process is not in Idle state
;                          because CPU calculation is done when the request
;                          processing has terminated;
;   last request memory  - the max amount of memory the last request consumed
;                          it's always 0 if the process is not in Idle state
;                          because memory calculation is done when the request
;                          processing has terminated;
; If the process is in Idle state, then informations are related to the
; last request the process has served. Otherwise informations are related to
; the current request being served.
; Example output:
;   ************************
;   pid:                  31330
;   state:                Running
;   start time:           01/Jul/2011:17:53:49 +0200
;   start since:          63087
;   requests:             12808
;   request duration:     1250261
;   request method:       GET
;   request URI:          /test_mem.php?N=10000
;   content length:       0
;   user:                 -
;   script:               /home/fat/web/docs/php/test_mem.php
;   last request cpu:     0.00
;   last request memory:  0
;
; Note: There is a real-time FPM status monitoring sample web page available
;       It's available in: /usr/share/php/7.4/fpm/status.html
;
; Note: The value must start with a leading slash (/). The value can be
;       anything, but it may not be a good idea to use the .php extension or it
;       may conflict with a real PHP file.
; Default Value: not set
;pm.status_path = /status

; The ping URI to call the monitoring page of FPM. If this value is not set, no
; URI will be recognized as a ping page. This could be used to test from outside
; that FPM is alive and responding, or to
; - create a graph of FPM availability (rrd or such);
; - remove a server from a group if it is not responding (load balancing);
; - trigger alerts for the operating team (24/7).
; Note: The value must start with a leading slash (/). The value can be
;       anything, but it may not be a good idea to use the .php extension or it
;       may conflict with a real PHP file.
; Default Value: not set
;ping.path = /ping

; This directive may be used to customize the response of a ping request. The
; response is formatted as text/plain with a 200 response code.
; Default Value: pong
;ping.response = pong

; The access log file
; Default: not set
;access.log = log/$pool.access.log

; The access log format.
; The following syntax is allowed
;  %%: the '%' character
;  %C: %CPU used by the request
;      it can accept the following format:
;      - %{user}C for user CPU only
;      - %{system}C for system CPU only
;      - %{total}C  for user + system CPU (default)
;  %d: time taken to serve the request
;      it can accept the following format:
;      - %{seconds}d (default)
;      - %{miliseconds}d
;      - %{mili}d
;      - %{microseconds}d
;      - %{micro}d
;  %e: an environment variable (same as $_ENV or $_SERVER)
;      it must be associated with embraces to specify the name of the env
;      variable. Some exemples:
;      - server specifics like: %{REQUEST_METHOD}e or %{SERVER_PROTOCOL}e
;      - HTTP headers like: %{HTTP_HOST}e or %{HTTP_USER_AGENT}e
;  %f: script filename
;  %l: content-length of the request (for POST request only)
;  %m: request method
;  %M: peak of memory allocated by PHP
;      it can accept the following format:
;      - %{bytes}M (default)
;      - %{kilobytes}M
;      - %{kilo}M
;      - %{megabytes}M
;      - %{mega}M
;  %n: pool name
;  %o: output header
;      it must be associated with embraces to specify the name of the header:
;      - %{Content-Type}o
;      - %{X-Powered-By}o
;      - %{Transfert-Encoding}o
;      - ....
;  %p: PID of the child that serviced the request
;  %P: PID of the parent of the child that serviced the request
;  %q: the query string
;  %Q: the '?' character if query string exists
;  %r: the request URI (without the query string, see %q and %Q)
;  %R: remote IP address
;  %s: status (response code)
;  %t: server time the request was received
;      it can accept a strftime(3) format:
;      %d/%b/%Y:%H:%M:%S %z (default)
;      The strftime(3) format must be encapsuled in a %{<strftime_format>}t tag
;      e.g. for a ISO8601 formatted timestring, use: %{%Y-%m-%dT%H:%M:%S%z}t
;  %T: time the log has been written (the request has finished)
;      it can accept a strftime(3) format:
;      %d/%b/%Y:%H:%M:%S %z (default)
;      The strftime(3) format must be encapsuled in a %{<strftime_format>}t tag
;      e.g. for a ISO8601 formatted timestring, use: %{%Y-%m-%dT%H:%M:%S%z}t
;  %u: remote user
;
; Default: "%R - %u %t \"%m %r\" %s"
;access.format = "%R - %u %t \"%m %r%Q%q\" %s %f %{mili}d %{kilo}M %C%%"

; The log file for slow requests
; Default Value: not set
; Note: slowlog is mandatory if request_slowlog_timeout is set
;slowlog = log/$pool.log.slow

; The timeout for serving a single request after which a PHP backtrace will be
; dumped to the 'slowlog' file. A value of '0s' means 'off'.
; Available units: s(econds)(default), m(inutes), h(ours), or d(ays)
; Default Value: 0
;request_slowlog_timeout = 0

; Depth of slow log stack trace.
; Default Value: 20
;request_slowlog_trace_depth = 20

; The timeout for serving a single request after which the worker process will
; be killed. This option should be used when the 'max_execution_time' ini option
; does not stop script execution for some reason. A value of '0' means 'off'.
; Available units: s(econds)(default), m(inutes), h(ours), or d(ays)
; Default Value: 0
;request_terminate_timeout = 0

; The timeout set by 'request_terminate_timeout' ini option is not engaged after
; application calls 'fastcgi_finish_request' or when application has finished and
; shutdown functions are being called (registered via register_shutdown_function).
; This option will enable timeout limit to be applied unconditionally
; even in such cases.
; Default Value: no
;request_terminate_timeout_track_finished = no

; Set open file descriptor rlimit.
; Default Value: system defined value
;rlimit_files = 1024

; Set max core size rlimit.
; Possible Values: 'unlimited' or an integer greater or equal to 0
; Default Value: system defined value
;rlimit_core = 0

; Chroot to this directory at the start. This value must be defined as an
; absolute path. When this value is not set, chroot is not used.
; Note: you can prefix with '$prefix' to chroot to the pool prefix or one
; of its subdirectories. If the pool prefix is not set, the global prefix
; will be used instead.
; Note: chrooting is a great security feature and should be used whenever
;       possible. However, all PHP paths will be relative to the chroot
;       (error_log, sessions.save_path, ...).
; Default Value: not set
;chroot =

; Chdir to this directory at the start.
; Note: relative path can be used.
; Default Value: current directory or / when chroot
;chdir = /var/www

; Redirect worker stdout and stderr into main error log. If not set, stdout and
; stderr will be redirected to /dev/null according to FastCGI specs.
; Note: on highloaded environement, this can cause some delay in the page
; process time (several ms).
; Default Value: no
;catch_workers_output = yes

; Decorate worker output with prefix and suffix containing information about
; the child that writes to the log and if stdout or stderr is used as well as
; log level and time. This options is used only if catch_workers_output is yes.
; Settings to "no" will output data as written to the stdout or stderr.
; Default value: yes
;decorate_workers_output = no

; Clear environment in FPM workers
; Prevents arbitrary environment variables from reaching FPM worker processes
; by clearing the environment in workers before env vars specified in this
; pool configuration are added.
; Setting to "no" will make all environment variables available to PHP code
; via getenv(), $_ENV and $_SERVER.
; Default Value: yes
;clear_env = no

; Limits the extensions of the main script FPM will allow to parse. This can
; prevent configuration mistakes on the web server side. You should only limit
; FPM to .php extensions to prevent malicious users to use other extensions to
; execute php code.
; Note: set an empty value to allow all extensions.
; Default Value: .php
;security.limit_extensions = .php .php3 .php4 .php5 .php7

; Pass environment variables like LD_LIBRARY_PATH. All $VARIABLEs are taken from
; the current environment.
; Default Value: clean env
;env[HOSTNAME] = $HOSTNAME
;env[PATH] = /usr/local/bin:/usr/bin:/bin
;env[TMP] = /tmp
;env[TMPDIR] = /tmp
;env[TEMP] = /tmp

; Additional php.ini defines, specific to this pool of workers. These settings
; overwrite the values previously defined in the php.ini. The directives are the
; same as the PHP SAPI:
;   php_value/php_flag             - you can set classic ini defines which can
;                                    be overwritten from PHP call 'ini_set'.
;   php_admin_value/php_admin_flag - these directives won't be overwritten by
;                                     PHP call 'ini_set'
; For php_*flag, valid values are on, off, 1, 0, true, false, yes or no.

; Defining 'extension' will load the corresponding shared extension from
; extension_dir. Defining 'disable_functions' or 'disable_classes' will not
; overwrite previously defined php.ini values, but will append the new value
; instead.

; Note: path INI options can be relative and will be expanded with the prefix
; (pool, global or /usr)

; Default Value: nothing is defined by default except the values in php.ini and
;                specified at startup with the -d argument
;php_admin_value[sendmail_path] = /usr/sbin/sendmail -t -i -f www@my.domain.com
;php_flag[display_errors] = off
;php_admin_value[error_log] = /var/log/fpm-php.www.log
;php_admin_flag[log_errors] = on
;php_admin_value[memory_limit] = 32M
```
