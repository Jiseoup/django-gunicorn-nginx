# How to setup Django with Gunicorn and Nginx in Ubuntu

* Ubuntu 20.04 LTS
* Python 3.8.10
* Django 4.1.2
* Gunicorn 20.1.0
* Nginx 1.18.0
* PostgreSQL 14.5

## Update & Upgrade before starting setup
```bash
$ sudo apt-get update
$ sudo apt-get upgrade
```

## Install Packages
```bash
$ sudo apt-get install python-is-python3 python3-pip python3-venv python3-dev
$ sudo apt-get install libpq-dev postgresql postgresql-contrib
```

## Setting python path
```bash
$ echo "export PATH=\$PATH:\$HOME/.local/bin" >> ~/.bashrc
$ source ~/.bashrc
```

## Create Directories for Project
```bash
$ mkdir project_dir
$ cd project_dir
```
```bash
$ mkdir project
$ mkdir sock
$ sudo chown {user_name}:www-data sock
```

## Create Virtual Environment
```bash
$ python -m venv venv
$ source venv/bin/activate
```

## Install pip modules in virtual environment
```bash
(venv) $ pip install django gunicorn
(venv) $ pip install psycopg2-binary
```

## Start Django Project
```bash
(venv) $ cd project
(venv) $ django-admin startproject conf .
```

## Create PostgreSQL Database and User
```bash
(venv) $ sudo -u postgres psql
```
```
postgres=# CREATE DATABASE project_db;
postgres=# CREATE USER project_user WITH PASSWORD 'password';

postgres=# ALTER ROLE project_user SET client_encoding TO 'utf8';
postgres=# ALTER ROLE project_user SET default_transaction_isolation TO 'read committed';
postgres=# ALTER ROLE project_user SET timezone TO 'UTC';

postgres=# GRANT ALL PRIVILEGES ON DATABASE project_db TO project_user;

postgres=# \q
```

## Adjusting Project Settings
```bash
(venv) $ vi conf/settings.py
```
```python
ALLOWED_HOSTS = ['*']
```
```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'NAME': 'project_db',
        'USER': 'project_user',
        'PASSWORD': 'password',
        'HOST': 'localhost',
        'PORT': '',
    }
}
```
```python
import os
STATIC_ROOT = os.path.join(BASE_DIR, 'static/')
```

## Completing Initial Setup
```bash
(venv) $ python manage.py makemigrations
(venv) $ python manage.py migrate
(venv) $ python manage.py createsuperuser
(venv) $ python manage.py collectstatic
```

## Open the port (Optional)
```bash
(venv) $ sudo ufw allow 8000
(venv) $ sudo ufw allow 8000/tcp
```

## Start Django Server
```bash
(venv) $ python manage.py runserver 0.0.0.0:8000
```

## Testing Gunicorn’s Ability to Serve the Project
```bash
(venv) $ gunicorn --bind 0.0.0.0:8000 conf.wsgi
```
```bash
(venv) $ deactivate
```

## Create Gunicorn Service
```bash
$ sudo vi /etc/systemd/system/gunicorn.service
```
```service
[Unit]
Description=gunicorn daemon
After=network.target

[Service]
User={user_name}
Group=www-data
WorkingDirectory=/home/{user_name}/project_dir/project
ExecStart=/home/{user_name}/project_dir/venv/bin/gunicorn \
          --workers 2 \
          --bind unix:/home/{user_name}/project_dir/sock/gunicorn.sock \
          conf.wsgi:application

[Install]
WantedBy=multi-user.target
```
```bash
$ sudo systemctl daemon-reload
$ sudo systemctl enable gunicorn
$ sudo systemctl start gunicorn
$ sudo systemctl status gunicorn
```

## Nginx Configuration
```bash
$ sudo apt install nginx
$ sudo vi /etc/nginx/site-available/project.conf
```
```conf
server {
    listen 80;
    server_name {your_ip_address};   # your ip or domain address
    
    location = /favicon.ico { access_log off; log_not_found off; }
    
    location /static/ {
        root /home/{user_name}/project_dir/project;
    }
    
    location / {
        include proxy_params;
        proxy_pass http://unix:/home/{user_name}/project_dir/sock/gunicorn.sock;
    }
}
```
```bash
$ sudo ln -s /etc/nginx/sites-available/project.conf /etc/nginx/sites-enabled/
$ sudo nginx -t   # test
$ sudo systemctl enable nginx
$ sudo systemctl restart nginx
```
