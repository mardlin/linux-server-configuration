# Linux server configuration

## Overview

This repository is a walkthrough of the steps required to securely configure an AWS hosted Ubuntu Linux server to serve a Python Flask application. In this case the application is my [Quotes Catalog](https://github.com/mardlin/quotes-catalog) application, which is currently visible at http://ec2-52-41-151-189.us-west-2.compute.amazonaws.com/

The server is accessible via ssh on port 2200 as 'grader' (iff you have the SSH key).

`$ ssh grader@52.41.151.189 -p 2200 -i ~/.ssh/udacity_grader`

## Software installed and configured

A list of the software involved in this configuration. (Linked to configuration details)

- Ubuntu 14.04.4 LTS
- SSH
- UFW (Uncomplicated Firewall)
- Apache2
- Mod-WSGI
- Python 2.7
- PostgreSQL 9.3
- Flask

## Configuration summary

### 1. Ubuntu configuration

#### Create a new user with sudo permissions 

Ref: [AskUbuntu](http://askubuntu.com/questions/7477/how-can-i-add-a-new-user-as-sudoer-using-the-command-line#7484) 

In our case the user will be named "grader":
```
$ sudo adduser grader
```

Create a new file with the same name as the user in `/etc/sudoers.d/`, and populate the file with:

```
grader ALL=(ALL:ALL) ALL
```

#### Update all currently installed packages

```
$ sudo apt-get update
$ sudo apt-get upgrade
```

#### Configure the local timezone to UTC
ref: [Unix Stackexchange](http://unix.stackexchange.com/a/110529)

```
$ sudo dpkg-reconfigure tzdata
```

This will bring up a GUI with regional options: 
    - Select "None of the above".
    - Select "UTC"


### 2. SSH
Ref: [AskUbuntu](http://askubuntu.com/questions/16650/create-a-new-ssh-user-on-ubuntu-server)

#### Add SSH key to server

Generate key on local machine

```
$ ssh-keygen
```

Append the contents of the public key (`keyname.pub`) into `~/.ssh/authorized_keys` in the user's home directory.

Set the appropriate permissions:

```
$ sudo chown -R username:username /home/grader/.ssh
$ sudo chmod 0700 ~/.ssh
$ sudo chmod 0600 ~/.ssh/authorized_keys
```
#### Update SSH configuration settings

```
$ sudo nano /etc/ssh/sshd_config
```

Modify the following settings to let our new user in, and provide some security through obscurity:
```
    Port 2200 
    PermitRootLogin no
    AllowUsers grader
```

$ sudo service ssh restart


Make sure you can still login using you ssh key:

```
$ ssh grader@52.41.151.189 -p 2200 -i ~/.ssh/udacity_grader
```

Update the `sshd_config` file to disable password auth thusly requiring keys for login

```
PasswordAuthentication no
```

Restart ssh: 

```
$ sudo service ssh restart
```

### 3. Uncomplicated Firewall (UFW)

#### Allow only incoming connections for SSH, HTTP, and NTP.
Ref: [Digital Ocean](https://www.digitalocean.com/community/tutorials/how-to-setup-a-firewall-with-ufw-on-an-ubuntu-and-debian-cloud-server)

Set the defaults:
```
$ sudo ufw default deny incoming
$ sudo ufw default allow outgoing 
$ sudo ufw default allow www # opens port 80
$ sudo ufw default allow ntp # opens port 123
$ sudo ufw default allow 2200/tcp # `allow ssh` would open port 22, not what we want here.
```

Activate UFW:

```
$ sudo ufw
```

### 3. Install Apache2
ref: [Digital Ocean](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)

```
$ sudo apt-get update
$ sudo apt-get install apache2
```

Visit server's IP address to verify that "it works".

### 4. Install and Enable mod_wsgi

```
$ sudo apt-get install libapache2-mod-wsgi python-dev
$ sudo a2enmod wsgi # enable mod_wsgi
```


### 5. Create a simple Flask app

```
$ cd /var/www/
$ mkdir testApp
$ cd testApp
$ mkdir testApp
$ mkdir static templates
```
We should now have the following directory structure:

```
|----testApp
|---------testApp
|--------------static
|--------------templates
```

create `__init__.py` in the inner most `testApp` folder with the following contents:

```
from flask import Flask
app = Flask(__name__)
@app.route("/")
def hello():
    return "Hello, I love Digital Ocean!"
if __name__ == "__main__":
    app.run()
```

### 5. Install Flask
In this section we install Flask, and configure Apache with mod_wsgi to serve a simple application.

Install virtualenv to isolate app dependencies from main server:

```
$ sudo pip install virtualenv
```

Create a virtual environment name 'venv':

```
$ sudo virtualenv venv
```

Run the virtual environment:

```
$ source venv/bin/activate
```
Install Flask inside venv:

```
$ sudo pip install Flask`
```
Check the file::

```
$ sudo python __init__.py 
```

If Flask is successfully installed, it will print `Running on http://localhost:5000/` or `Running on http://127.0.0.1:5000/`.

#### Create an Apache2 configuration file to serve the app

```
$ sudo nano /etc/apache2/sites-available/testApp.conf
```
Paste the following into the file, modifying the site address and directories as appropriate

```
<VirtualHost *:80>
    ServerName 52.41.151.189
    ServerAdmin admin@52.41.151.189
    WSGIScriptAlias / /var/www/testApp/testApp.wsgi
    <Directory /var/www/testApp/testApp/>
        Order allow,deny
        Allow from all
    </Directory>
    Alias /static /var/www/testApp/testApp/static
    <Directory /var/www/testApp/testApp/static/>
        Order allow,deny
        Allow from all
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

note: The default for `${APACHE_LOG_DIR}` is `/var/log/apache2`.

Enable the virtual host
```
$ sudo a2ensite testApp
```

#### Create the WSGI file

In testApp.conf, we have this line: 
```       
        WSGIScriptAlias / /var/www/FlaskApp/flaskapp.wsgi
```

Which points Apache to a file needed to serve our Flask app, so we'll create that file:

```
$ sudo nano /var/www/FlaskApp/flaskapp.wsgi
```

Then insert the following text: 

```
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/testApp/")

from testApp import app as application
application.secret_key = 'YOUR_SECRET_KEY'
```

You can easily generate a good secret key in the python interpreter:
```
>>> import os
>>> os.urandom(24)
'\xfd{H\xe5<\x95\xf9\xe3\x96.5\xd1\x01O<!\xd5\xa2\xa0\x9fR"\xa1\xa8'
```
ref: [Flask Docs](http://flask.pocoo.org/docs/0.11/quickstart/#sessions)

#### Restart Apache2

```
$ sudo service apache2 restart
```
You should now be able to visit your app at the `ServerName` specified in the Virtual Host configuration. 

Now that we know we can serve a Flask app using Apache and mod_wsgi, we can move on with installing my database driven quotes catalog app. 

### 6. Install and configure Postgres
ref: [Digital Ocean](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-14-04)

Install PostgreSQL:     

```
$ sudo apt-get install postgresql postgresql-contrib
```

#### Ensure remote connection are disallowed:

Review the Postgres configuration file:

```
$ sudo nano /etc/postgresql/9.3/main/pg_hba.conf
```

These are the defaults: 

```
# TYPE  DATABASE        USER            ADDRESS                 METHOD
local   all             postgres                                peer
# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
# IPv6 local connections:
host    all             all             ::1/128                 md5
```

All connections are either `local` or else limit the IP address to the local machine, so 


While here, also add the following _above_ these rules:
```
host   catalog          catalog         127.0.0.1/32            md5
```

This will allow our `catalog` user to connect to the `catalog` database on behalf of the application. 

Create the `catalog` user:

```
$ sudo adduser catalog
```

Create a `catalog` role in postgres:

```
$ sudo -i -u postgres       # switch to postgres user
$ createuser --interactive  # when called as postgres, creates a postgres role. "Name it catalog"
$ createdb catalog
$ exit                      # leave the postgres user session
```

Set the `catalog` role's password in postgres:

```
$ sudo -i -u catalog        # switch to catalog user
$ psql                      # open the postgres command line tool
catalog=# \password         # enter a password.
catalog=# \q                # exit psql
```

### 7. Install git
ref 

```
$ sudo apt-get update
$ sudo apt-get install git
```

### 8. Install the catalog application

Clone the github repo:
``` 
$ cd /var/www
$ git clone https://github.com/mardlin/quotes-catalog.git
```

The app uses SQLAlchemy as an ORM to the database. The original application used an SQLite database, so we'll update the connection string for a postgres database (ref: [SQLAlchemy](http://docs.sqlalchemy.org/en/latest/core/engines.html?highlight=create_engine#postgresql))

```
engine = create_engine('postgresql://catalog:fullstack@127.0.0.1/catalog')
```

Install required third party libraries:
```
$ sudo pip install Flask 
$ sudo pip install Flask-SeaSurf # CSRF protection
$ sudo pip install oauth2client 
$ sudo pip install requests 
$ sudo pip install SQLAlchemy
```

Set up and populate the database:

```
$ sudo python database_setup.py
$ sudo python populate_database.py
```

#### Create the wsgi file

In the top level director of the application, create a file named `catalog.wsgi`. Our Apache site config file will point to this file, which in turn points to our `runserver.py` file. 

Within this file add the following:

```
#!/usr/bin/python
import sys
import logging

logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/quotes-catalog")
sys.path.insert(0,"/var/www/quotes-catalog/catalog/")

from runserver import app as application
```

Note: a more common last line might be `from __init__ import app as application`, however
this application used a slightly different structure to enable splitting the Flask app's router logic across multiple files. ref: ([Flask Docs](http://flask.pocoo.org/docs/0.11/patterns/packages/))


#### Create a new apache virtual host config file

```
$ sudo nano /etc/apache2/sites-available/catalog.conf
```

The file should have the same contents as `testApp.conf`, but with the directory paths updated to point to the new catalog app: 

```
<VirtualHost *:80>
    ServerName 52.41.151.189
    ServerAdmin admin@52.41.151.189

    WSGIScriptAlias / /var/www/quotes-catalog/catalog.wsgi
    <Directory /var/www/quotes-catalog/catalog>
        Order allow,deny
        Allow from all
    </Directory>
    Alias /static /var/www/quotes-catalog/catalog/static
    <Directory /var/www/quotes-catalog/catalog/static/>
        Order allow,deny
        Allow from all
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

Disable the test application's virtual host:

``` 
$ sudo a2dissite testApp.conf
```

Enable the catalog application's virtual host:

``` 
$ sudo a2ensite catalog.conf
```

Restart Apache:

```
$ sudo service apache2 restart
```

Verify the site is working: http://ec2-52-41-151-189.us-west-2.compute.amazonaws.com/

#### Update OAuth2 Settings

Visit https://console.developers.google.com/apis/credentials/oauthclient, and select the application. 

Under "Authorized JavaScript origins" add the base URL. Under "Authorized redirect URIs" add the OAuth2 callback URL. 

Test OAuth2 login.

{{celebratory GIF}}










