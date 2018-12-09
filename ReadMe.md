# Linux Configuration

The following stpes were performed to setup the linux server.

## Environment

Created a AWS Lightstail instance with Ubuntu 18.04.1 LTS.

## User management

The following steps were done:

~~~
adduser grader
visudo
Added the line grader ALL=(ALL:ALL) ALL
~~~

## SSH Key-pair

Generated keys on local machine using ssh-keygen

Saved the private key under ~/.ssh

On the remote server, following steps were done:

~~~
su - grader
mkdir .ssh
touch .ssh/authorized_keys
vim .ssh/authorized_keys
~~~

Saved the public key in this file and given the following permissions

~~~
chmod 700 .ssh
chmod 644 .ssh/authorized_keys
~~~

Connected to the remote server with the following command

~~~
ssh grader@13.232.243.59 -p 2200 -i ~/.ssh/linuxCourse
~~~

## Updated the system

All the packages were updated by executing below step

~~~
sudo apt-get update
sudo apt-get upgrade
~~~

## SSH port modification

Opened the config file:

~~~
sudo vim /etc/ssh/sshd_config
~~~

Changed to Port 2200.

Changed PermitRootLogin from without-password to no.

Restart SSH Service:

~~~
sudo /etc/init.d/ssh restart
~~~


## Firewall configuration changes

Opened the ports using following commands:

~~~
sudo ufw allow 2200/tcp
sudo ufw allow 80/tcp
sudo allow 123/udp
~~~

Restarted SSH service.

Verified the config changes by:

~~~
sudo /etc/init.d/ssh restart
sudo uwf status
~~~

## Apache and mod-wsgi setup

Installed Apache HTTP Server

    sudo apt-get update
    sudo apt-get install apache2

Installed and enabled mod-wsgi (for Python3)

    sudo apt-get install libapache2-mod-wsgi-py3
    sudo a2enmod wsgi

Installed PIP3

    sudo apt-get update
    sudo apt-get -y install python3-pip

Created a flask test app under /var/www/test/test.py.
A corresponding wsgi script is also created, /var/www/test/app.wsgi

Content of app.wsgi

~~~
#!/usr/bin/python3

import sys
import logging
sys.path.insert(0,"/var/www/test/")

from test import app as application
~~~

Content of test.py

~~~
#/usr/bin/python3

from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello...'

if __name__ == '__main__'
    app.run(host='0.0.0.0', port=5000)
~~~

Create VirtualHost config file in /etc/apache2/sites-available/test.conf

~~~
<VirtualHost *:80>
ServerName PUBLIC_IP
WSGIScriptAlias / /var/www/test/app.wsgi
DocumentRoot /var/www/test
<Directory /var/www/test/>
    Order deny,allow
    Allow from all
</Directory>

ErrorLog ${APACHE_LOG_DIR}/error.log
LogLevel warn
CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
~~~

Changed the owner of the project folder, enabled the site and restarted the Apache service

~~~
sudo chown www-data:www-data /var/www/test -R
sudo a2ensite test.conf
sudo systemctl restart apache2
~~~

Successfully accessed the test app using public IP through browser.
Disabled the site

~~~
sudo a2dissite test.conf
~~~

## Deployed CatalogApp

As a next step, deployed the catalog app.

Created the folder structure:

~~~
/var/www/FlaskApp/CatalogApp
~~~

Created the wsgi script file: 

~~~
/var/www/FlaskApp/catalog.wsgi
~~~

Downloaded catalog source files from AWS S3 into CatalogApp.

~~~
/var/www/FlaskApp/CatalogApp/views.py
/var/www/FlaskApp/CatalogApp/databasemodels.py
/var/www/FlaskApp/CatalogApp/static/*
/var/www/FlaskApp/CatalogApp/templates/*
~~~

Installed all the dependencies 

~~~
sudo pip3 install flask
sudo pip3 install httplib2
sudo pip3 install request
sudo pip3 install flask_httpauth
sudo pip3 install oauth2client
sudo pip3 install sqlalchemy
sudo pip3 install passlib
~~~

Tested successfully using curl after launching it locally.

~~~
python3 databasemodels.py
python3 views.py
~~~

Created catalog.wsgi in /var/www/FlaskApp/ so that CatalogApp and .wsgi are at the same folder.

~~~
#!/usr/bin/python3

import sys
import logging
import os
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/FlaskApp/")
sys.path.insert(0,"/var/www/FlaskApp/CatalogApp")

from CatalogApp import g_app as application
application.secret_key = os.urandom(24)
~~~

Renamed CatalogApp/views.py to

~~~
CatalogApp/__init__.py
~~~

Created a VirtualHost config file /etc/apache2/sites-available/catalog_fin.conf and enabled the site

~~~
  <VirtualHost *:80>
      ServerName PUBLIC-IP-ADDRESS
      ServerAdmin admin@PUBLIC-IP-ADDRESS
      WSGIScriptAlias / /var/www/FlaskApp/catalog.wsgi
      <Directory /var/www/FlaskApp/CatalogApp/>
          Order allow,deny
          Allow from all
      </Directory>
      Alias /static /var/www/catalog/catalog/static
      <Directory /var/www/catalog/catalog/static/>
          Order allow,deny
          Allow from all
      </Directory>
      ErrorLog ${APACHE_LOG_DIR}/catalog_final_error.log
      LogLevel warn
      CustomLog ${APACHE_LOG_DIR}/access.log combined
  </VirtualHost>
~~~
~~~
sudo chown www-data:www-data /var/www/FlaskApp -R
sudo a2ensite catalog_fin.conf
sudo systemctl reload apache2
~~~

Could not test successfully with browser due to the following error.

~~~
sqlalchemy.exc.OperationalError: (OperationalError) unable to open database file 
~~~

After extensive research, it seemed an issue related to sqlite. 
So I was left with no option other than porting CatalogApp to Postgresql

~~~
sudo apt-get install postgresql postgresql-contrib
sudo vi /etc/postgresql/10/main/pg_hba.con
sudo vi databasemodels.py
~~~

Modified the create_engine argument as below.

~~~
create_engine('postgresql://postgres:password@localhost/itemcatalog_ps')
~~~

Set password for the psql user postgres.

Made below change in /etc/postgres/10/pg_hba.conf to get rid of the error "Peer authentication failed for user postgres"

~~~
local   all             postgres                                peer
to 
local   all             postgres                                md5
~~~

Created database by executing below command

~~~
python3 databasemodels.py
~~~

Re-enabled the site again and restarted Apache

Successfully accessed the app using browser.

## Get Google OAuth working

Get the hostname for my public IP address

Updated hostname in the VirtualHost config

~~~
sudo nano /etc/apache2/sites-available/catalog_fin.conf
~~~

Added the following line

~~~
ServerAlias ec2-13-232-243-59.ap-south-1.compute.amazonaws.com
~~~

Re-enabled the site

~~~
sudo a2ensite catalog_fin.conf
~~~

Updated Authorized JavaScript origins and Authorized redirect URIs in Google API console

Bingo!!!..now the site is accessible through browser using hostname and IP address.

~~~
IP Address  :   13.232.243.59
Hostname    :   ec2-13-232-243-59.ap-south-1.compute.amazonaws.com
SSH         :   ssh grader@13.232.243.59 -p 2200 -i <private_key_file>
~~~

That's it... :)

## Open issue

Static files are not being served :(
I have struggled a lot to find a solution but no success.

## References

A million sites on the internet, to name a few:

https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps#step-five-%E2%80%93-create-the-wsgi-file

https://teamtreehouse.com/community/python-35-flask-virtualenv-and-apache2

https://stackoverflow.com/questions/30642894/getting-flask-to-use-python3-apache-mod-wsgi

https://devops.profitbricks.com/tutorials/deploy-a-flask-application-on-ubuntu-1404/

https://teamtreehouse.com/community/python-35-flask-virtualenv-and-apache2

https://github.com/squadran2003/flask-bootstrap-apache-template

https://github.com/stueken/FSND-P5_Linux-Server-Configuration

https://medium.com/@esteininger/python-3-5-flask-apache2-mod-wsgi3-on-ubuntu-16-04-67894abf9f70

Last but not least, Udacity classroom and mentors.



