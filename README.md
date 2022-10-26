# How to setup Django with Gunicorn and Nginx in Ubuntu
This document is created with reference to the corresponding [tutorial](https://www.digitalocean.com/community/tutorials/how-to-set-up-django-with-postgres-nginx-and-gunicorn-on-ubuntu-22-04).  

## 1. System Configuration
I checked that all of the following works correctly in the two system configurations.  
One is a local environment, one is a cloud environment.
* **Local**
    * `Ubuntu 20.04` + `Python 3.8.10` + `PostgreSQL 12.12`
* **Oracle Cloud**  
    * `Ubuntu 22.04` + `Python 3.10.6` + `PostgreSQL 14.5`
    
The configuration below is the same for all environments.
* Django 4.1.2
* Gunicorn 20.1.0
* Nginx 1.18.0

## 2. Initial Environment Configuration
### Update & Upgrade before starting setup
```bash
$ sudo apt-get update
$ sudo apt-get upgrade
```

### Install packages
This document uses PostgreSQL as the database.
```bash
$ sudo apt-get install python-is-python3 python3-pip python3-venv python3-dev
$ sudo apt-get install libpq-dev postgresql postgresql-contrib
```

### Setting python path
```bash
$ echo "export PATH=\$PATH:\$HOME/.local/bin" >> ~/.bashrc
$ source ~/.bashrc
```

### Create directories for project
```bash
$ mkdir project_dir
$ cd project_dir
```
```bash
$ mkdir project
$ mkdir sock
$ sudo chown {user_name}:www-data sock
```

## 3. Configuring a Virtual Environment
Now, create a virtual environment and install the required pip modules.
### Create virtual environment
```bash
$ python -m venv venv
$ source venv/bin/activate
```

## Install pip modules in virtual environment
```bash
(venv) $ pip install django gunicorn
(venv) $ pip install psycopg2-binary
```

## 4. Django Settings
### Start Django project
```bash
(venv) $ cd project
(venv) $ django-admin startproject conf .
```

### Create PostgreSQL database and user
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

### Adjusting project settings
```bash
(venv) $ vi conf/settings.py
```
Enter the server address or domain that can connect to django.  
Addresses are separated by commas.
```python
ALLOWED_HOSTS = ['*']
```
Now modify the database information.  
As mentioned earlier, this document uses PostgreSQL.
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
Add the contents below under `STATIC_URL = 'static/'`.
```python
import os
STATIC_ROOT = os.path.join(BASE_DIR, 'static/')
```

### Completing initial setup
Now we can use the `manage.py` script to migrate the database schema to the PostgreSQL database.
```bash
(venv) $ python manage.py makemigrations
(venv) $ python manage.py migrate
```
Create a project manager using the commands below.
```bash
(venv) $ python manage.py createsuperuser
```
Static contents (js, css ...) can be collected using the command below.
```bash
(venv) $ python manage.py collectstatic
```

### Open the port (Optional)
If you are using cloud services, please follow the instructions below.  
Open 8000 ports in the cloud, and the server also generates exceptions for 8000 ports.
```bash
(venv) $ sudo ufw allow 8000
(venv) $ sudo ufw allow 8000/tcp
```

### Start Django server
Verify that the django server is working well.
```bash
(venv) $ python manage.py runserver 0.0.0.0:8000
```

## 5. Gunicorn Settings
### Testing Gunicorn’s ability to serve the project
First, make sure that you can serve the application through the Gunicorn test.
```bash
(venv) $ gunicorn --bind 0.0.0.0:8000 conf.wsgi
```
If it works well, exits the virtual environment.
```bash
(venv) $ deactivate
```

### Create Gunicorn service
Create a Gunicorn service file in the path below.
```bash
$ sudo vi /etc/systemd/system/gunicorn.service
```
Write the following content.
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

### Start the Gunicorn service
Start the Gunicorn service, and check the status.
```bash
$ sudo systemctl daemon-reload
$ sudo systemctl enable gunicorn
$ sudo systemctl start gunicorn
$ sudo systemctl status gunicorn
```

## 6. Nginx Settings
### Install Nginx and create configuration
Install Nginx and create an Nginx configuration file in the path below.
```bash
$ sudo apt install nginx
$ sudo vi /etc/nginx/site-available/project.conf
```
Write the following content.
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

### Start the Nginx
Attach a symbolic link to the Nginx configuration file and test it to see if it works.
```bash
$ sudo ln -s /etc/nginx/sites-available/project.conf /etc/nginx/sites-enabled/
$ sudo nginx -t   # test
```
The results should be as follows.
```bash
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Restart the Nginx, and check the status.
```bash
$ sudo systemctl enable nginx
$ sudo systemctl restart nginx
```

## 7. Concluding
Now you can access your Django application without having to access the development server.  
Instead, you need to open a firewall for normal traffic on port 80.  
You may also remove exceptions for port 8000.
```bash
$ sudo ufw allow 'Nginx Full'
$ sudo ufw delete allow 8000
$ sudo ufw delete allow 8000/tcp
```

Now enjoy your jango application!
