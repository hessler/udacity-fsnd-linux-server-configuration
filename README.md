# Udacity FSND - Linux Server Configuration
Repository for the Udacity Linux Server Configuration project.

## Access Information
- IP Address: `52.40.91.240`
- SSH Port: `2200`
- Web App URL: http://ec2-52-40-91-240.us-west-2.compute.amazonaws.com/


## Installed Software
_Rather than list out all installed software and packages here, please refer to the various code snippets in the Configuration Changes section below._


## Configuration Changes
### Add user _grader_ with sudo permissions

Add new user named **grader** (password: _LetMeIn!_):
```
adduser grader
```

Grant sudo privileges by creating **grader** in `sudoers.d`, and adding appropriate permissions:
```
sudo nano /etc/sudoers.d/grader
grader ALL=(ALL) NOPASSWD:SHUTDOWN_CMDS
```

Add public key authentication by switching go new **grader** user, creating `authorized_keys` file in newly created `.ssh` directory and pasting in contents of `udacity_key.rsa` into file, then restricting permissions on the `authorized_keys`:
```
su - grader
mkdir .ssh
chmod 700 .ssh
nano .ssh/authorized_keys
chmod 600 .ssh/authorized_keys
```

### Update Installed Packages
```
sudo apt-get update
sudo apt-get upgrade
```

### Configure Local Timezone to UTC

Follow directions in Terminal from the command below. Select _"None of the above"_, then _"UTC"_. Confirm the local time is same as universal time.
```
dpkg-reconfigure tzdata
```

### Change SSH Port

Edit SSH config file, changing `Port 22` to `Port 2200`, then reload SSH:
```
sudo nano /etc/ssh/sshd_config
sudo reload ssh
```

### Uncomplicated Firewall (UFW) Configuration

Set defaults to deny incoming and allow outgoing, allow ssh, www, and ntp, and enable and confirm UFW:
```
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow www
sudo ufw allow ntp
sudo ufw enable
sudo ufw status
```

### Install & Configure Git, PostgreSQL, and Apache

Initial installation of packages:
```
sudo apt-get update
sudo apt-get install apache2
sudo apt-get install postgresql postgresql-contrib
sudo apt-get install git
```

Set global Git configuration items:
```
git config --global user.name "Anthony Hessler"
git config --global user.email "anthony.hessler@covenanteyes.com"
```

Add user **catalog**, create database, and revoke/grant appropriate access:
```
sudo -u postgres createuser --pwprompt catalog
sudo -u postgres createdb -O catalog catalog
sudo -u postgres psql -d catalog -c "REVOKE ALL ON SCHEMA public FROM public;"
sudo -u postgres psql -d catalog -c "GRANT ALL ON SCHEMA public TO catalog;"
/etc/init.d/postgresql restart
sudo service apache2 restart
```

### Set up & Serve Catalog App

Add new directory for `catalog` and create `.htaccess` file with contents:
```
cd /var/www/
sudo mkdir catalog
cd catalog
sudo nano .htaccess
```

**.htaccess** contents: 
```
RedirectMatch 404 /\.git
```

Clone Catalog app into separate directory, then move contents over to newly created `catalog` directory. This is necessary due to the way in which the original Catalog app was set up and committed to GitHub. _(Note: The last line simply lists the contents of the `catalog` directory, as a way to confirm everything was copied over as intended.)_
```
cd /var/www/
sudo mkdir fsnd-p3
cd fsnd-p3
sudo git clone https://github.com/hessler/fullstack-nanodegree-vm.git
cd ..
sudo mv fsnd-p3/fullstack-nanodegree-vm/vagrant/catalog/catalog/ catalog/
cd catalog
ls -al
```

>_Note: At this point, I had to make some miscellaneous updates to the Catalog App files, again due to how the app was originally set up vs. how it needed to be configured to work here._

Install necessary packages for the Catalog App:
```
sudo apt-get update
sudo apt-get install python-pip
sudo apt-get install python-psycopg2
sudo apt-get install python-flask
sudo apt-get install python-sqlalchemy
sudo apt-get install python-oauth2client (or, run: sudo pip install oauth2client)
sudo apt-get install libapache2-mod-wsgi
sudo pip install Flask-SQLAlchemy
```

Create configuration file for new catalog app:
```
cd /etc/apache2/sites-available/
sudo nano catalog.conf
```

Contents of configuration file:
```
<VirtualHost *:80>
    ServerName 52.40.91.240
    ServerAdmin anthony.hessler@covenanteyes.com
    ServerAlias ec2-52-40-91-240.us-west-2.compute.amazonaws.com
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
    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel info
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

Create WSGI file:
```
cd /var/www/catalog
sudo nano catalog.wsgi
```

Contents of WSGI file:
```
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/catalog/")
from catalog import app as application
application.secret_key = '$up3r$3cr3tK3y'
```

At this point, I updated the JSON paths in `catalog/views.py` to include the full path, rather than a relative path:
```
FACEBOOK_JSON = '/var/www/catalog/catalog/client_secrets_facebook.json'
GOOGLE_JSON = '/var/www/catalog/catalog/client_secrets_google.json'
```

Enable the new catalog site and disable the default one:
```
sudo a2ensite catalog
sudo a2dissite 000-default
sudo service apache2 reload
```


## Third-party Resources
Below is a list of tutorials, articles, forum discussions, and other resources that were used for researching and completing this project. They are broken down into sections, but the list is in no particular order of importance.

### Server Setup & Configuration
- [Initial Server Setup with Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-14-04)
- [Initial Server Setup with Ubuntu 12.04 (Step 5 - Configure SSH)](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-12-04)
- [How to Install Linux, Apache, MySQL, PHP (LAMP) stack on Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-install-linux-apache-mysql-php-lamp-stack-on-ubuntu-14-04)
- [Add/Delete Users on Ubuntu](https://www.digitalocean.com/community/tutorials/how-to-add-and-delete-users-on-an-ubuntu-14-04-vps)
- [How to Set Up a Firewall with UFW on Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu-14-04)
- [Linux Server Configuration](https://github.com/robertavram/Linux-Server-Configuration)
- [How to Use Vagrant for Local Web Development](http://blog.osteel.me/posts/2015/01/25/how-to-use-vagrant-for-local-web-development.html)
- [Time Zone in Ubuntu](https://help.ubuntu.com/community/UbuntuTime#Using_the_Command_Line_.28terminal.29)

### Flask
- [How to Deploy a Flask Application on an Ubuntu VPS](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)

### GitHub
- [Git (Ubuntu Server Guide)](https://help.ubuntu.com/lts/serverguide/git.html)
- [How to Install Git on Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-install-git-on-ubuntu-14-04)
- [How do you clone a git repository into a specific folder?](http://stackoverflow.com/questions/651038/how-do-you-clone-a-git-repository-into-a-specific-folder)

### PostgreSQL
- [How to Secure PostgreSQL on an Ubuntu VPS](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps)
- [PostgreSQL createuser](https://www.postgresql.org/docs/9.1/static/app-createuser.html)
- [Postgresql and TCP/IP connections on port 5432](http://www.railszilla.com/postgresql-tcpip-connections-port-5432/coffee-break)
- [How Do I Enable remote access to PostgreSQL database server?](http://www.cyberciti.biz/tips/postgres-allow-remote-access-tcp-connection.html)

### Udacity Forum Discussions
- [Markedly underwhelming and potentially wrong resource list for P5](https://discussions.udacity.com/t/markedly-underwhelming-and-potentially-wrong-resource-list-for-p5/8587)
- [P5 How I got through it](https://discussions.udacity.com/t/p5-how-i-got-through-it/15342/)
- [Project 5 Resources](https://discussions.udacity.com/t/project-5-resources/28343)
- [Git Bash command to upload Project to server](https://discussions.udacity.com/t/git-bash-command-to-upload-project-to-server/28671/)
- [List of items in the directory instead of displaying site](https://discussions.udacity.com/t/list-of-items-in-the-directory-instead-of-displaying-site/35423)
- [Running Apache problem](https://discussions.udacity.com/t/running-apache-problem/35719/)
- [Help needed. WSGI deployment error](https://discussions.udacity.com/t/help-needed-wsgi-deployment-error/18242/)

