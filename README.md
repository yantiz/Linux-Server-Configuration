# Linux Server Configuration 
This project is to configure an Apache HTTP Server to host my previous Flask application named [Item Catalog](https://github.com/yantiz/Item-Catalog).

### Instruction on how to remotely access my web server via SSH

Download the private key `grader_rsa` from this repository and move it into `~/.ssh` of your local machine.
Then type `ssh -i ~/.ssh/grader_rsa grader@35.167.117.224 -p 2200` to ssh into the server as the user of 'grader'.

### Instruction on how to connect to my app's webpage through browsers 

[Click me](http://ec2-35-167-117-224.us-west-2.compute.amazonaws.com) to see my app currently being hosted on my web server.

# The setup procedure I followed: 

## Step 1: Launch your Virtual Machine with my Udacity account
`https://www.udacity.com/account#!/development_environment`

## Step 2: SSH into my server
```
mv ~/Downloads/udacity_key.rsa ~/.ssh/
chmod 600 ~/.ssh/udacity_key.rsa
ssh -i ~/.ssh/udacity_key.rsa root@35.167.117.224
```

## Step 3: Create a new user named grader and its password is set to 'udacity'
`sudo adduser grader`

## Step 4: Give the grader the permission to sudo and set up its SSH keys
```
sudo usermod -aG sudo grader
su grader
```

When using grader user to issue a sudo command, I received the following warning: `sudo: unable to resolve host ip-10-20-36-105`.
To fix this, the hostname was added to the loopback address `/etc/hosts` by inserting `127.0.1.1 ip-10-20-36-105` at the second line of the file

```
ssh-keygen -t rsa
cp ~/.ssh/grader_rsa.pub ~/.ssh/authorized_keys
```
copy `grader_rsa` into `~/.ssh` of my host machine
run `chmod 600 ~/.ssh/grader_rsa` on my host machine

## Step 5: Update all currently installed packages
```
sudo apt-get update
sudo apt-get upgrade
```

## Step 6: Change the SSH port from 22 to 2200 and disable root login
```
sudo vim /etc/ssh/sshd_config

Port 2200
DenyUsers root
AllowUsers grader

PermitRootLogin no
```

## Step 7: Configure the ufw to only allow incoming requests for SSH(port 2200), HTTP and NTP.
```
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2200/tcp
sudo ufw allow www
sudo ufw allow ntp
sudo ufw enable
```

## Step 8: Configure the local timezone to UTC
`sudo dpkg-reconfigure tzdata`

## Step 9: Install and configure Apache to serve a Python mod_wsgi application
```
sudo apt-get install apache2
sudo apt-get install libapache2-mod-wsgi python-dev
sudo a2enmod wsgi
```

## Step 10: Install and configure PostgreSQL
- Goal 1: Create a new user named catalog that has limited permissions to your catalog application database.
- Goal 2: Do not allow remote connections.

```
sudo apt-get install postgresql
sudo -u postgres psql
CREATE DATABASE item_catalog;
CREATE USER catalog WITH PASSWORD 'catalog';
GRANT ALL PRIVILEGES ON DATABASE item_catalog TO catalog;
\q
```

By default, remote connection to PostgreSQL is not allowed. However, for security concerns, we should 
double check by typing `sudo vim /etc/postgresql/9.3/main/pg_hba.conf` to inspect the content of the file.
```
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             postgres                                peer
local   all             all                                     peer

# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
# IPv6 local connections:
host    all             all             ::1/128                 md5
```
As we can see, the first two security lines specify "local" as the scope that they apply to. 
This means they are using Unix/Linux domain sockets.

The second two declarations are remote, but if we look at the hosts that they apply to 
(127.0.0.1/32 and ::1/128), we see that these are interfaces that specify the local machine.

Therefore, the default settings have already prevented remote connections.

# Step 11: Set up my Item Catalog app to make it functional within my Apache web server

- Clone my Flask app from github to my Apache web server
```
sudo apt-get install git
sudo git clone https://github.com/yantiz/Item-Catalog.git /var/www/catalog/catalog
```

- Download necessary dependencies for my app
```
sudo apt-get install python-psycopg2
sudo apt-get install libpq-dev
sudo pip install -r /var/www/catalog/catalog/dependencies.txt
```

- Configure and enable a new virtual host
```
sudo vim /etc/apache2/sites-available/catalog.conf

<VirtualHost *:80>
        ServerName ec2-35-167-117-224.us-west-2.compute.amazonaws.com
        ServerAdmin admin@mywebsite.com
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
        LogLevel warn
        CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>

sudo a2ensite catalog
```

- Create and configure catalog.wsgi
```
sudo vim /var/www/catalog.wsgi

#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, '/var/www/catalog/catalog')

from catalog import app as application
```

- Prevent .git directory from being publicly accessed via browsers
```
sudo vim /var/www/catalog/catalog/.git/.htaccess

<Directory .git>
    order allow,deny
    deny from all
</Directory>
```

- Restart Apache server to apply changes made

`sudo service apache2 restart`

# References
[https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)
[https://httpd.apache.org/docs/2.2/configuring.html](https://httpd.apache.org/docs/2.2/configuring.html)
[https://code.google.com/archive/p/modwsgi/wikis/ConfigurationDirectives.wiki](https://code.google.com/archive/p/modwsgi/wikis/ConfigurationDirectives.wiki)
[http://stackoverflow.com/questions/6142437/make-git-directory-web-inaccessible](http://stackoverflow.com/questions/6142437/make-git-directory-web-inaccessible)
