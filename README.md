# PgAmin4-Nginx-uwsgi

## Installation guide for PgAmin4 on Ubuntu 20.04 using Nginx

Install ubuntu packages
```
sudo apt install python3-pip build-essential python3-dev libssl-dev libffi-dev python3-virtualenv
```

Setup python environment
```
mkdir pgadmin
cd pgadmin
virtualenv pgadmin
source pgadmin/bin/activate

sudo pip3 install pgadmin4
sudo pip3 install uwsgi
```

Create local pgAmin config file
```
sudo vim /usr/local/lib/python3.8/dist-packages/pgadmin4/config_local.py
```

Add the following to config_local.py
```
LOG_FILE = '/var/log/pgadmin/pgadmin.log'
SQLITE_PATH = '/var/lib/pgadmin/pgadmin.db'
SESSION_DB_PATH = '/var/lib/pgadmin/sessions'
STORAGE_DIR = '/var/lib/pgadmin/storage'
SERVER_MODE = True
ALLOW_SAVE_PASSWORD = True
```

Create PgAmin superuser
```
sudo python3.8 /usr/local/lib/python3.8/dist-packages/pgadmin4/setup.py
```

Create uwsgi ini file
```
sudo vim /etc/uwsgi/pgadmin.ini
```
```
[uwsgi]
uid             = www-data
gid             = www-data
chdir           = /usr/local/lib/python3.8/dist-packages/pgadmin4 
wsgi-file       = /usr/local/lib/python3.8/dist-packages/pgadmin4/pgAdmin4.wsgi
master          = true
processes       = 1
socket          = /tmp/pgadmin.sock
chmod-socket    = 664
vacuum          = true
```

Change the owner of /var/log/pgadmin and /var/lib/pgadmin to www-data
```
sudo chown www-data:www-data /var/lib/pgadmin/pgadmin.db
sudo chown -R www-data:www-data /var/log/pgadmin/
sudo chown -R www-data:www-data /var/lib/pgadmin/
```

To test uwsgi
```
sudo /usr/local/bin/uwsgi --ini /etc/uwsgi/pgadmin.ini
```

Create uwsgi.service in /etc/systemd to auto start in on boot
```
sudo vim /etc/systemd/system/uwsgi.service
```
```
[Unit]
Description=uWSGI service unit
After=syslog.target

[Service]
ExecStart=/usr/local/bin/uwsgi --ini /etc/uwsgi/pgadmin.ini
ExecReload=/bin/kill -HUP $MAINPID
ExecStop=/bin/kill -INT $MAINPID
Restart=always
Type=notify
StandardError=syslog
NotifyAccess=all
KillSignal=SIGQUIT
StandardOutput=/var/log/uwsgi/my.log
StandardError=/var/log/uwsgi/error.log

[Install]
WantedBy=multi-user.target
```

Configure Nginx
```
upstream pgadmin {
      server unix:/tmp/pgadmin.sock;
}
server {
    listen 5050;
    server_name  _;
    charset      utf-8;
    client_max_body_size 75M;
    location / {
        include uwsgi_params;
        uwsgi_pass  pgadmin;
    }
}
#server{
#    listen          443 ssl;
#    server_name     _;
#    charset      utf-8;
#    ssl_certificate      /etc/uwsgi/pgadmin.crt;
#    ssl_certificate_key  /etc/uwsgi/pgadmin.key;
#    location / {
#        include uwsgi_params;
#        uwsgi_pass  pgadmin;
#    }
#}
```
```
upstream pgadmin_upstream {
  server unix:/tmp/pgadmin.sock;
}

server {
      listen 80;
      server_name pgadmin.example.org;
      access_log off;
      error_log off;
      ## redirect http to https ##
      return 301 https://$server_name$request_uri;
}

# HTTPS server
server {
  listen 443 ssl http2;
  server_name pgadmin.example.org;

  ssl_certificate      /path/to/my/ssl/server.crt;
  ssl_certificate_key   /path/to/my/ssl//server.key;
  ssl_session_timeout  5m;

  error_page   500 502 503 504  /50x.html;
    # Only allow local network
  allow 192.168.1.0/24; 
  deny all;
  
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
  ssl_ciphers EECDH AES128:RSA AES128:EECDH AES256:RSA AES256:EECDH 3DES:RSA 3DES:EECDH RC4:RSA RC4:!MD5;
  ssl_prefer_server_ciphers on; 

  location / {
    uwsgi_pass pgadmin_upstream;
    include uwsgi_params;
    uwsgi_modifier1 30;
  }
}
```
