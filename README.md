# Linux Server Configuration Project
- [Linux Server Configuration Project](#linux-server-configuration-project)
        - [About](#about)
    - [Server Overview](#server-overview)
    - [Basic server setup](#basic-server-setup)
        - [Set up new Ubuntu Server instance on Amazon Lightsail](#set-up-new-ubuntu-server-instance-on-amazon-lightsail)
        - [SSH into the server](#ssh-into-the-server)
        - [Update all installed packages](#update-all-installed-packages)
        - [Configure the local timezone to UTC](#configure-the-local-timezone-to-utc)
        - [Suggestion: create an instance snapshot](#suggestion-create-an-instance-snapshot)
    - [Security configuration](#security-configuration)
        - [Change the SSH port from 22 to 2200 and disable remote root login](#change-the-ssh-port-from-22-to-2200-and-disable-remote-root-login)
        - [Firewall configuration](#firewall-configuration)
    - [Adding a user](#adding-a-user)
        - [Create new user **'grader'** with sudo access](#create-new-user-grader-with-sudo-access)
        - [Set up SSH login with keys for grader user](#set-up-ssh-login-with-keys-for-grader-user)
    - [Set up database and app dependencies](#set-up-database-and-app-dependencies)
        - [Apache and mod_wsgi installation](#apache-and-modwsgi-installation)
        - [PostgreSQL installation](#postgresql-installation)
        - [PostgreSQL configuration: catalog user and database](#postgresql-configuration-catalog-user-and-database)
        - [Set up Linux user for database access](#set-up-linux-user-for-database-access)
        - [Git Installation and Item Catalog setup](#git-installation-and-item-catalog-setup)
        - [Setup and enble virtual host](#setup-and-enble-virtual-host)
        - [WSGI Configuration](#wsgi-configuration)
        - [Disable default Apache startpage and enable the Itme Catalog](#disable-default-apache-startpage-and-enable-the-itme-catalog)
        - [Edit the database path](#edit-the-database-path)
        - [Initialize database](#initialize-database)
        - [SSL Setup](#ssl-setup)
    - [Sources](#sources)
### About
This is the final project in Udacity's [Full Stack Web Developer Nanodegree](https://www.udacity.com/course/full-stack-web-developer-nanodegree--nd004).<br>
The previously built [Item Catalog](https://github.com/laudep/ItemCatalog) web app will be hosted on a Ubuntu server running on an [Amazon Lightsail](https://aws.amazon.com/lightsail/) instance.<br>
The server details and setup/configuration steps taken are listed below.

## Server Overview
| Attribute                	| Value                                                         	|
|--------------------------	|---------------------------------------------------------------	|
| **Public IP**            	| 35.156.161.50                                                 	|
| **SSH port**             	| 2200                                                          	|
| **Item Catalog app URL** 	| https://ec2-35-156-161-50.eu-central-1.compute.amazonaws.com/ 	|

## Basic server setup
### Set up new Ubuntu Server instance on Amazon Lightsail
1. Log in on AWS ([create an account](https://aws.amazon.com/resources/create-account/) if needed)
2. Click **Create instance** button on the home page
3. Select the **Linux/Unix** platform and choose **OS Only** with **Ubuntu** as blueprint
4. Select an instance plan
5. Name your instance
6. Click **Create** button and wait for the instance to start up

### SSH into the server
If your browser supports it you can connect using Amazon's browser-based ssh client by clicking the **'Connect using SSH'** button.  
Alternatively follow the steps below in order to SSH into the server from your local machine.
1. Download your private key from the [**'SSH keys'** tab on the account page](https://lightsail.aws.amazon.com/ls/webapp/account/keys) on Amazon Lightsail.  
   The default filename is _LightsailDefaultPrivateKey-[region].pem_
2. Create a new file in the ~/.ssh folder on your local machine.<br>
   Copy the private key file contents into it and save.
    ```console
   $ touch ~/.ssh/lightsail_server.rsa
   $ nano ~/.ssh/lightsail_server.rsa
   ```
3. Update the file permission to owner only.
    ```console
    $ chmod 600 ~/.ssh/lightsail_server.rsa
    ```
4. SSH into the instance.
     ```console
     $ ssh -i ~/.ssh/lightsail_server.rsa ubuntu@35.156.161.50
     ```


### Update all installed packages
```console
$ sudo apt-get update
$ sudo apt-get upgrade
$ sudo apt-get dist-upgrade
```
If the message `*** System restart required ***` appears, reboot the server using `sudo reboot`.

### Configure the local timezone to UTC
```console
$ sudo dpkg-reconfigure tzdata
```
Choose **None of the above** followed by **UTC**.

### Suggestion: create an instance snapshot
[Create an instance snapshot](https://lightsail.aws.amazon.com/ls/docs/en/articles/lightsail-how-to-create-a-snapshot-of-your-instance) before making changes on SSH and firewall configuration.  
This ensures that the instance doesn't need to be set up again from scratch when something goes wrong.

## Security configuration
###  Change the SSH port from 22 to 2200 and disable remote root login
1. Open the configuration file.
2. Change the port number from **22** to **2200**.
3. Set `PermitRootLogin` to `no`.
4. Restart SSH service.
```console
$ sudo nano /etc/ssh/sshd_config
$ sudo service ssh restart
```
3. Open the **Networking** tab of the Lightsail instance page.  
   Add custom port TCP 2200 to the **Firewall** category.  
   Delete default SSH port 22.
4. From now port 2200 must be specified when connecting through SSH.  
   Loging in using Amazon's browser-based ssh client by clicking the 'Connect using SSH' button **will no longer work**.
```console
$ ssh -i ~/.ssh/lightsail_server.rsa ubuntu@35.156.161.50 -p 2200
```

###  Firewall configuration
1. Make sure ufw firewall is disabled
2. Deny all incoming connections by default
3. Accept all outgoing connections be default
4. Allow incoming traffic for **SSH** on port 2200
5. Allow incoming traffic for **HTTP** on port 80
6. Allow traffic for **NTP** on port 123
7. Close port 22
8. Double check before enabling the firewall
9. Enable firewall
10. Check new firewall status
11. Open the **Networking** tab of the Lightsail instance page.  
   Add custom ports **UDP 123** for NTP.

```console
$ sudo ufw default deny incoming
$ sudo ufw default allow outgoing
$ sudo ufw allow 2200/tcp
$ sudo ufw allow www
$ sudo ufw allow ntp
$ sudo ufw deny 22
$ sudo ufw show added
$ sudo ufw enable
$ sudo ufw status
```


## Adding a user
### Create new user **'grader'** with sudo access
1. Create new user account named 'grader'
2. Give sudo access by creating the file `/etc/sudoers.d/grader` containing the text `grader ALL=(ALL) NOPASSWD:ALL`.
```console
$ sudo adduser grader
$ sudo touch /etc/sudoers.d/grader
$ sudo nano /etc/sudoers.d/grader
grader ALL=(ALL) NOPASSWD:ALL
```

### Set up SSH login with keys for grader user
* On your local machine
    * Create an SSH key pair for the grader user in the `~/.ssh` folder.  
      Read its content to and copy it to clipboard.
    ```console
    $ ssh-keygen -f ~/.ssh/grader_key
    $ cat ~/.ssh/grader_key
    ```
* On the server instance
    * As the grader user create file `.ssh/authorized_keys`.
    * Copy the above key contents into it and save.
    * Set correct permissions on the `.ssh` folder and the `.ssh/authorized_keys` file.
    * Verify password login is disabled in file `/etc/ssh/sshd_config`:
    `PasswordAuthentication` should be set to `no`.
    * Restart SSH service.
    ```console
    $ su grader
    $ cd
    $ mkdir .ssh
    $ touch .ssh/authorized_keys
    $ nano .ssh/authorized_keys
    Copy key contents and save.
    $ chmod 700 .ssh
    $ chmod 644 .ssh/authorized_keys
    $ sudo nano /etc/ssh/sshd_config
    $ sudo service ssh restart
    ```
Verify grader login:
```console
$ ssh -i ~/.ssh/grader_key -p 2200 grader@35.156.161.50
```


## Set up database and app dependencies
### Apache and mod_wsgi installation
```console
$ sudo apt-get install apache2
$ sudo apt-get install libapache2-mod-wsgi python-dev
$ sudo a2enmod wsgi
$ sudo service apache2 restart
```
Open http://35.156.161.50/ in a web browser.  
An **Apache2 Ubuntu Default Page** should open if the Apache installation was succesful.

### PostgreSQL installation
```console
$ sudo apt-get install postgresql postgresql-contrib
```
Make sure remote connections are being blocked.
```console
$ sudo nano /etc/postgresql/9.5/main/pg_hba.conf
```

### PostgreSQL configuration: catalog user and database
```console
$ sudo su - postgres
$ psql
# CREATE ROLE catalog WITH PASSWORD 'password';
# ALTER ROLE catalog WITH LOGIN;
# ALTER USER catalog CREATEDB;
# CREATE DATABASE catalog WITH OWNER catalog;
# \c catalog
# REVOKE ALL ON SCHEMA public FROM public;
# GRANT ALL ON SCHEMA public TO catalog;
# \q
$ exit
```

### Set up Linux user for database access
```console
$ sudo adduser catalog
$ sudo visudo
```
Add `catalog ALL=(ALL:ALL) ALL`.  
Save and exit.
```console
$ sudo su - catalog
$ createdb catalog
$ exit
```


### Git Installation and Item Catalog setup
```console
$ sudo apt-get install git
$ sudo mkdir /var/www/catalog
$ cd /var/www/catalog
$ sudo git clone https://github.com/laudep/ItemCatalog catalog
$ sudo chown -R ubuntu:ubuntu catalog/
$ cd catalog
$ mv application.py __init__.py
$ nano __init__.py
```
Change the last line (`app.run(host='0.0.0.0', port=8000)`) into `app.run()`.  
Save and exit the file.
Change `application` into `__init__` in `create_sample_catalog.py`.


###Installation of Flask, SQLAlchemy and other dependencies
```console
$ sudo apt-get install python-pip
$ sudo pip install httplib2
$ sudo pip install requests
$ sudo pip install --upgrade oauth2client
$ sudo pip install flask
$ sudo pip install sqlalchemy
$ sudo apt-get install libpq-dev
$ sudo pip install psycopg2
```

### Setup and enble virtual host
```console
$ sudo touch /etc/apache2/sites-available/catalog.conf
$ sudo nano /etc/apache2/sites-available/catalog.conf
 <VirtualHost *:80>
		ServerName 35.156.161.50
		ServerAdmin admin@35.156.161.50
		WSGIScriptAlias / /var/www/catalog/catalog.wsgi
		<Directory /var/www/catalog/catalog/>
            Require all granted
		</Directory>
		Alias /static /var/www/catalog/catalog/static
		<Directory /var/www/catalog/catalog/static/>
			Require all granted
		</Directory>
		ErrorLog ${APACHE_LOG_DIR}/error.log
		LogLevel warn
		CustomLog ${APACHE_LOG_DIR}/access.log combined
   </VirtualHost>
```
Save and exit file.
```console
$ sudo a2ensite catalog
$ sudo service apache2 reload
```

### WSGI Configuration
```console
$ sudo touch /var/www/catalog/catalog.wsgi
$ sudo nano /var/www/catalog/catalog.wsgi
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/catalog/")

from catalog import app as application
application.secret_key = 'secret_key'
```
Save and exit file.
```console
$ sudo service apache2 reload
```

### Disable default Apache startpage and enable the Itme Catalog
```console
$ sudo a2dissite 000-default.conf
$ sudo a2ensite catalog.conf
$ sudo service apache2 reload
```

### Edit the database path
Replace all `engine = create_engine` lines in `__init__.py` and `database_setup.py` with  
`engine = create_engine('postgresql://catalog:[database_password]@localhost/catalog')`

### Initialize database
```console
$ sudo python create_sample_catalog.py
$ sudo service apache2 reload
```

### SSL Setup
Optionally, SSL can be enabled on the server by following the [excellent DigitalOcean guide](https://www.digitalocean.com/community/tutorials/how-to-create-a-self-signed-ssl-certificate-for-apache-in-ubuntu-16-04).


## Sources
- [Lightsail Documentation | Amazon Web Services](https://lightsail.aws.amazon.com/ls/docs/en/all)
- [Configuring Linux Web Servers | Udacity](https://www.udacity.com/course/configuring-linux-web-servers--ud299)
- [Configuration Files - Apache HTTP Server | Apache Software Foundation](http://httpd.apache.org/docs/current/configuring.html)
- [How To Secure PostgreSQL on an Ubuntu VPS | DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps)
- [How To Create a Self-Signed SSL Certificate for Apache in Ubuntu 16.04 | DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-create-a-self-signed-ssl-certificate-for-apache-in-ubuntu-16-04)