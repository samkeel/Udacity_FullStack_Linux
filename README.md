# Udacity Full Stack Project 5: Linux configuration
Taking a baseline installtion of a Linux distribution on a virtual machine and modifying it to host web applications, including updates, securing it from numerous attack vectors and installing and configuring web and database servers.
## Server details
* IP address: ~~13.211.66.185~~
* SSH Port: 2200
* URL: ~~http://13.211.66.185.xip.io/~~ **Site taken down after grading.**

# Summary of changes made
* Create new user 'Grader' and add to Sudo group
* Configure OpenSSH to allow grader to login using private SSH key
* Configure UncomplicatedFirewall (UFW) to allow traffic on ports 123, 80 and 220
* Configure NTP server for server time management. 
* Clone existing Github repo
  * Rename and repurpose it to use a different database and run on a lightsail server, not a virtual server.
* Create new catalog PostgreSQL account and add records from python scripts.
* Modify the Apache default config file to run the WSGI application.

# Software and ubuntu packages installed:
* Openssh Server
* NTP
* Apache2
* PostgreSQL
* GIT
* mod_wsgi
* Python
* virtualenv
* Flask
* httplib2
* sqlalchemy
* psycopg2
* sqlalchemy_utils
* requests

# References
* [Lightsail Docs](https://lightsail.aws.amazon.com/ls/docs/all)
* [Ubuntu server guide](https://help.ubuntu.com/lts/serverguide/index.html)
* [Stackoverflow ](http://www.stackoverflow.com/)
* [Learning PostgreSQL](https://www.packtpub.com/big-data-and-business-intelligence/learning-postgresql)
  * By Salahaldin Juba, Achim Vannahme, Andrey Volkov
* [Mastering Ubuntu Server](https://www.packtpub.com/networking-and-servers/mastering-ubuntu-server)
  * By Jay LaCroix
* [Ubuntu Server Cookbook](https://www.packtpub.com/networking-and-servers/ubuntu-server-cookbook)
  * By Uday R. Sawant
* [Ubuntu Server Essentials](https://www.packtpub.com/networking-and-servers/ubuntu-server-essentials)
  * By Abdelmonam Kouka
* [Digital Ocean tutorials](https://www.digitalocean.com/community/tutorials)


# Walkthrough

## Configuring server

### User setup
Create new user named grader: 
```
sudo adduser grader
```
Append user to sudo group: 
```
sudo usermod -aG sudo grader
```
#### OpenSSH config
Install and setup Openssh server 
```
sudo apt-get install openssh-server
sudo nano /etc/ssh/sshd_config
```
Edit port to 2200 save and exit config file.
Restart server.
```
sudo /etc/init.d/ssh restart
```
### Give grader sudo access
```
sudo visudo
```
Add new line under root
```
grader ALL=(ALL:ALL) ALL
```
### Add ports
```
sudo ufw allow ssh
sudo ufw allow 2200/tcp
sudo ufw allow www
sudo ufw allow 123
sudo ufw deny 22
sudo ufw enable
```
### Switch user to grader and create authorized keys
Open a copy of GitBash locally and run `ssh-keygen` save the contents to default directory as grader_key

Open grader_key.pub and copy the contents `cat ~/.ssh/grader_key.pub`

On the server as root, switch user to grader and add ssh login.
```
su grader
cd
mkdir /home/grader/.ssh
sudo touch .ssh/authorized_keys
sudo nano .ssh/authorized_keys
```
Paste contents of grader_key.pub for the ssh key.
```
chown grader:grader /home/grader/.ssh
sudo chmod 700 .ssh
sudo chmod 644 .ssh/authorized_keys
id grader
sudo nano /etc/ssh/sshd_config
```
In sshd_config uncomment line 'AuthorizedKeysFile %h/.ssh/authorized_keys' to enable ssh login.
## Disable root user
Change line 'PermitRootLogin prohibit-password' to 'PermitRootLogin no' to disable root account.

Save and restart the server.
```
sudo systemctl restart ssh
```

grader can now log onto the server using a command line utility such as GitBash
```
ssh grader@13.211.66.185 -p 2200 -i  ~/.ssh/grader_key
```
## Install and Configure NTP time server
Install NTP
```
sudo apt-get install ntp
```
Check NTP status
```
/etc/init.d/ntp status
```
If the server is not set to UTC, set the time zone 
```
sudo timedatectl set-timezoneUTC
```
Verify the timezone has been set correctly and enable NTP in the UFW
```
timedatectl
sudo ufw disable
sudo ufw allow ntp
sudo ufw enable
```
## Install and configure PostgreSQL
Install apache2
```
sudo apt-get install apache2
```
Install mod_wsgi
```
sudo apt-get install libapache2-mod-wsgi
```
Install postgreSQL
```
Sudo apt-get install postgresql-contrib
```
Update all components
```
sudo apt-get update
sudo apt-get upgrade
```

### Configure PostgreSQL
Disable remote access
```
cd /etc/postgresql/9.5/main
sudo nano pg_hba.conf
```
```
#local   all             postgres                                peer
#local   all             all                                     peer
host    all             all             127.0.0.1/32            md5
host    all             all             ::1/128                 md5
```
Comment out the local access methods following digital ocean tutorial [How To Secure PostgreSQL on an Ubuntu VPS](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps)

## Create new PSQL user
```
sudo su - postgres
psql
CREATE USER catalog WITH PASSWORD ‘catalog’;
ALTER USER catalog CREATEDB;
CREATE DATABASE catalog WITH OWNER catalog;
\c catalog
REVOKE ALL ON SCHEMA public FROM public;
GRANT ALL ON SCHEMA public TO catalog;
\q
```
## Install Git

```
sudo apt-get install git
```
Install virtual environment and clone existing github project.
```
sudo apt-get install python-dev
sudo a2enmod wsgi
cd /var/www
sudo mkdir catalog
cd catalog
sudo git clone https://github.com/vectors36/Udacity_FullStack_Catlogue_CRUD.git
sudo mv Udacity_FullStack_Catlogue_CRUD catalog
Cd catalog
```
Install Flask and dependant packages
```
sudo apt-get install python-pip
sudo pip install virtualenv
sudo virtualenv venv
source venv/bin/activate
sudo pip install Flask
sudo pip install httplib2 
sudo pip install oauth2client 
sudo pip install sqlalchemy 
sudo pip install psycopg2 
sudo pip install sqlalchemy_utils 
sudo pip install requests

```
## Edit the Apache default config file and redirect .git requests to 404 error
```
sudo nano /etc/apache2/sites-enabled/000-default.conf
<VirtualHost *:80>
        ServerName 13.211.66.185
        ServerAlias ec2-13-211-66-185.ap-southeast-2.compute.amazonaws.com
        ServerAdmin admin@13.211.66.185
        WSGIScriptAlias / /var/www/catalog/catalog.wsgi
        <Directory /var/www/catalog/catalog/>
                Order allow,deny
                Allow from all
        </Directory>
        Alias /static /var/www/catalog/catalog/static
        <Directory /var/www/catalog/catalog/static/>
                Order allow,deny
                Allow from all
        </Directory>
        RedirectMatch 404 /\.git
</VirtualHost>
```
Create the .wsgi file
```
cd /var/www/catalog
Sudo nano catalog.wsgi

#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/catalog/")
from catalog import app as application
application.secret_key = ‘client_secret'
```
Rename app.py to \_\_init__.py

Edit \_\_init__.py
```
sudo nano __init__.py
```
Edit client_secrets.json reference from relative to fixed address:
```
CLIENT_ID = json.loads(
    open('/var/www/catalog/catalog/client_secrets.json', 'r').read())['web']['client_id']
APPLICATION_NAME = "movie_catalogues"
```
In program start remove virtual machine run command and replace with app.run()
```
# Program start
if __name__ == '__main__':
#    app.secret_key = 'super_secret_key'
#    app.debug = True
#    app.run(host='0.0.0.0', port=5000)
    app.run()
```
Change engine refrences from sqlite to postgresql in \_\_init__.py, database_setup.py and popdataset.py
```
- engine = create_engine('sqlite:///movies.db')
+ engine = create_engine('postgresql://catalog:catalog@localhost/catalog')

```
Run database_setup.py to create the database and popdataset.py to populate the database with preset information.
```
python database_setup.py
python popdataset.py
```
Restart the server
```
sudo apache2ctl restart
```
## Google oAuth2 workaround
Google oAuth2 does not support static IP address redirection, it requres a named domain, something ending with a .com or .net. Not wanting to pay for a named service with Route 53 for test code. A work around is to use the free wildcard DNS server [xip.io](http://xip.io). Open [Google developers console](https://console.developers.google.com) and modify the Authorized JavaScript origins to be the IP address with .xip.io at the end.
> http://13.211.66.185.xip.io

Update the Authorized redirect URI's to the server EC2 addresses 
> http://ec2-13-211-66-185.ap-southeast-2.compute.amazonaws.com/login

> http://ec2-13-211-66-185.ap-southeast-2.compute.amazonaws.com/gconnect

Update client_secrets.json with the new JSON codes provided by Google.
```
sudo nano client_secrets.json
```
Open site and confirm it works:
> ~~http://13.211.66.185.xip.io/~~ **Site taken down after grading.**
