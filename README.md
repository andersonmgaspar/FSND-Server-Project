# FSND-Server-Project

## Intro

Guidelines to configure a linux server in Amazon Lightsail and load a python app to run on this server.

## Server Definitions and Configurations

### Create an AWS Lightsail instance

- Create or Log-in into an AWS account [https://aws.amazon.com/](https://aws.amazon.com/);
- Create a new Lightsail Instance [https://lightsail.aws.amazon.com/ls/webapp/create/instance](https://lightsail.aws.amazon.com/ls/webapp/create/instance)
  - Select a plataform: Linux/Unix;
  - Select a blueprint: OS Only - Ubuntu 16.04 LTS;
  - Choose a instance plan (I chose the cheaper one ;))
  - Define a identification for the instance;
  - Create and profit (or pay the Amazon);


### Updating repos

First step to secure the server is update the packages and apps:

```
sudo apt-get update
sudo apt-get upgrade
sudo apt-get dist-upgrade
```

Another requirement of this project was to configure timezone to UTC:
```
sudo timedatectl set-timezone UTC
```
test typing: `date`

### New User and ssh key

Creating a new user named grader:
```
sudo adduser grader

#grant sudo to grader, add it to Sudoers
sudo usermod -aG sudo grader

```
Another requirement was to login with a public / secret key system, for this I create a local secret_key on Git Bash using the commmand:
```
 ssh-keygen
```
After this command you have to type the name of the key (grader-key) and a passphrase (I left passphrase blank).
Get the local key content, with this command:
`cat PATH-TO-KEY/.ssh/grader-key.pub`

And I sent the key to the reviewer in the project "Notes to Reviewer" field

To deploy public key on developement enviroment, we have to do the following steps:
```
#Create a .ssh directory on the grader home
sudo mkdir /home/grader/.ssh

#Give ownership to grader
sudo chown grader:grader /home/grader/.ssh

#to create a authorized_keys file and edit it
sudo touch /home/grader/.ssh/authorized_keys
#In the next command I added the public key on authorized_keys
sudo nano /home/grader/.ssh/authorized_keys

#Set the permissions to folder and files
sudo chmod 700 /home/grader/.ssh
sudo chmod 644 /home/grader/.ssh/authorized_keys

```

### SSH Configuration and access

First step is to Access the sshd_config file typing:
`sudo nano /etc/ssh/sshd_config`
Change the following parameters:
Port 2200
PasswordAuthentication no
PermitRootLogin no

After this, restart the ssh service
`sudo service ssh restart`

Access the server with Git Bash using the private key:
```
 ssh -i C:/Users/ander/.ssh/grader-key grader@54.160.14.106 -p 2200
 ssh -i PATHTOKEY/grader-key grader@54.160.14.106 -p 2200
```

### Firewall Configuration

The following commands set the firewall security of the server:

```
#Check the firewall status
sudo ufw status

#Block all incoming connections on all ports by default
sudo ufw default deny incoming

#Allow outgoing connection on all ports
sudo ufw default allow outgoing

#Allow incoming connection for SSH on port 2200 (REQUIREMENT)
sudo ufw allow 2200/tcp

#Allow incoming connections for HTTP on port 80 (REQUIREMENT)
sudo ufw allow www

#to allow incoming connection for NTP on port 123 (REQUIREMENT)
sudo ufw allow ntp

#Enable the firewall (ATTENTION Here, check the connections)
sudo ufw enable

#Check the status of the firewall
sudo ufw status
```


## App Configuration

### Database Configuration

Run the following commands to install and configure the PostgreSQL Database
```
sudo apt-get install postgresql
sudo apt-get install python-psycopg2
sudo apt-get install libpq-dev
```

After the installation I set a new user to the database:
`sudo su - postgres`


Using the postgres username, run `psql` to set some settings on PostgreSQL:

Create user:
```
CREATE USER grader;
ALTER ROLE grader WITH PASSWORD 'grader';
```
Grant user access to create database: `ALTER USER grader CREATEDB;`
Create database: `CREATE DATABASE catalog WITH OWNER grader;`
Revoke Schema access: `REVOKE ALL ON SCHEMA public FROM public;`
Grant user access: `GRANT ALL ON SCHEMA public TO grader;`
Check for remote connection settings here: `sudo nano /etc/postgresql/9.5/main/pg_hba.conf`

Quit `psql` command line by typing: `\q` and enter
Type `exit` to exit from the username "postgres"

### Apache install and configuration

Install Apache
`sudo apt-get install apache2`

Install mod_wsgi
`sudo apt-get install libapache2-mod-wsgi`

Restart Apache with
`sudo apache2ctl restart`


### Catalog App files and Configuration

First check for the git installation: `sudo apt-get install git`
Go to the /var/www directory: `cd /var/www`
Clone the repository using: ` git clone https://github.com/andersonmgaspar/Item-Catalog-Project-4.git`
Move the ./Item-Catalog-Project-4/vagrant/catalog to /var/www/catalog/catalog

Rename views.py to __init__.py using `sudo mv views.py __init__.py`
This is necessary for the WSGI Configuration

Change the app.cfg file in the app folder, updating the database to postgres: `sudo nano app.cfg`

```
SQLALCHEMY_DATABASE_URI = 'postgresql://grader:grader@localhost/catalog'
DEBUG = False
```

And change the init_db.py and models.py create_engine function:
```
create_engine('postgresql://grader:grader@localhost/catalog')
```

Install pip: `sudo apt-get install python-pip`
Get the requirements using pip: `sudo -H pip install -r requirements.txt`

Initialize the initial database: `sudo python init_db.py`


###Configure and Enable a New Virtual Host

Create catalog.conf to edit:
sudo nano /etc/apache2/sites-available/catalog.conf
```
<VirtualHost *:80>
        ServerName 54.160.14.106
        ServerAdmin andersonmgaspar@gmail.com
        ServerAlias 54.160.14.106.xip.io
        WSGIScriptAlias / /var/www/catalog/catalog.wsgi
        <Directory /var/www/catalog/catalog/>
                Order allow,deny
                Allow from all
        </Directory>
        ErrorLog ${APACHE_LOG_DIR}/error.log
        LogLevel warn
        CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

Enable the virtual host with the following command:

`sudo a2ensite ItemCatalog`


Restart Apache: `sudo apache2ctl restart`


Create the app.wsgi file

```
import sys

sys.path.insert(0,"/var/www/catalog/catalog")

from __init__ import app as application
application.secret_key = "\x15\x92u~_\xb6\xd12,\xe3\xb6'L\xa1\x87"

```



### Running the project


For Google OAuth2 it necessary to give the host name, so it was used the magic domain (http://xip.io/)
Check the g_cliente_secrets.json in the /var/www/catalog/catalog (CLIENT_SECRET is IMPORTANT):
```
{"web":{"client_id":"PUT_HERE_THE_GOOGLE_CLIENT_ID","project_id":"PUT_HERE_THE_PROJECT_ID","auth_uri":"https://accounts.google.com/o/oauth2/auth","token_uri":"https://oauth2.googleapis.com/token","auth_provider_x509_cert_url":"https://www.googleapis.com/oauth2/v2/certs","client_secret":"PUT_HERE_THE CLIENT_SECRET","redirect_uris":["http://54.160.14.106.xip.io/gconnect","http://54.160.14.106.xip.io/login"],"javascript_origins":["http://54.160.14.106.xip.io/"]}}
```

Restart Apache one more time to finish the configurations:
```
$ sudo apache2ctl restart
```

Now open the Browser and open the pray, I mean, open the page http://54.160.14.106.xip.io/
:D


## Supporting Materials

debug Apache2 server with the following command: `sudo tail -100 /var/log/apache2/error.log`  

Flask WSGI on Apache Tutorial:  
https://blog.ekbana.com/deploying-flask-application-using-mod-wsgi-bdf59174a389

Udacity

Some Github repositories that helped me to develop this project:  
[/rccmodena](https://github.com/rccmodena/FSND_linux_server_configuration)  
[/lucianobarauna](https://github.com/lucianobarauna/udacityproj_linuxserver)  
[/andrevst](https://github.com/andrevst/fsnd-p6-linux-server-configuration)
