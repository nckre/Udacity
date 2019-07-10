# Linux Server Configuration
### Udacity Fullstack Nanodegree - Project 3
_______________________
## Introduction
### What is this project about?
This project is linked to the 5th module of the Udacity Fullstack Nanodegree, the "Deploying to Linux Servers" course. The course teaches the basics of Linux & how to set up and secure a Linux server on the web. The goal and final project of the Nanodegree is to get a web application running live on a secure web server.

This ```README.md``` covers how the application from the 2nd project of the same nanodegree (available in this [repository](https://github.com/nckre/Udacity-FullstackND-Project2-ItemCatalog)) was deployed on an [Amazon Lightsail](https://lightsail.aws.amazon.com/) hosted Ubuntu Linux server.

### How is this document structured?
This ```README.md``` broadly follows the 5 steps described in Udacity's "Project Details" which are necessary to pass the project:
1. Get a server
2. Secure the server
3. Give access to a ```grader``` user
4. Prepare deployment of application
5. Deploy application to server
_______________________
## Get a server
This part had 2 requirements:
1. Start an Ubuntu Linux server instance on Amazon Lightsail.
2. SSH into the server
### Start an Ubuntu Server
I followed Udacity's recommendation and registered on [Amazon Lightsail](https://lightsail.aws.amazon.com/). After verification, I created an Ubuntu 16.04 instance in AWS Frankfurt data center.
![Lightsail Instance Setup](images/lightsail.png)
The server is currently (July 2019) available under the public IP ```18.197.158.41``` and the host name ```ec2-18-197-158-41.eu-central-1.compute.amazonaws.com```. Since AWS only provides one free month of hosting, the application will not be available post summer 2019.
### SSH into the server
For Linux-based instances, Lightsail uses Secure SHell (SSH) to connect to an server instance. SSH uses a key pair (a public key and a private key) to match the remote server to an authorized user. Lightsail's [account page](https://lightsail.aws.amazon.com/ls/webapp/account/keys) offers a download option for the default SSH key pair. The file name is based on the region so that my instance of the file was called ```LightsailDefaultKey-eu-central-1```. I use this name in the following steps:
1. Move file ```LightsailDefaultKey-eu-central-1``` from the local machine's ```Download``` folder to the  ```.ssh``` folder and rename it to ```udacity_key.rsa```.
Shell command: ```$ mv ~/Downloads/LightSailDefaultKey-eu-central-1 ~/.ssh/udacity_key.rsa```$
2. Change the access permission to the file to let only the owner write and read with the change mode 600 flag.
Shell command: ```chmod 600 ~/.ssh/udacity_key.rsa```
3. SSH into the server with the default ubuntu user & the server's public IP address. Use the -i option flag to signal that it's an integer.
Shell command: ```ssh -i ~/.ssh/udacity_key.rsa ubuntu@18.197.158.41```

At this point, the command prompt should change to ```ubuntu@18.197.158.41:~$``` (depending on prompt customizations)
_______________________
## Secure the server
This part had 3 requirements:
1. Update all currently installed packages.
2. Change the SSH port from 22 to 2200. Make sure to configure the Lightsail firewall to allow it.
3. Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123).

### Update packages
To update the currently installed packages, 2 commands were run:
1. Update the list of available packages and their versions
Shell command: ```sudo apt-get update```
2. Upgrade to newer versions of packages:
Shell command: ```sudo apt-get upgrade```

### Change SSH Port
Disclaimer by Udacity: "*When changing the SSH port, make sure that the firewall is open for port 2200 first, so that you don't lock yourself out of the server. When you change the SSH port, the Lightsail instance will no longer be accessible through the web app 'Connect using SSH' button. The button assumes the default port is being used. There are instructions on the same page for connecting from your terminal to the instance. Connect using those instructions and then follow the rest of the steps.*"
To change the SSH port 5 steps are needed:
1. Edit the ```sshd_config``` file inside the host specific configuration files ```/etc/ssh/sshd_config```.
Shell command: ```/sudo nano /etc/ssh/sshd_config```
2. Inside the file change the port number from the default ```22``` to ```2200``` and disable root login by setting ```PermitRootLogin``` to ```no```
3. Save and exit the file (i.e. type ```CTRL+X```, confirm changes with ```y```, confirm name with ```Enter```).
4. Restart the SSH connection
Shell command: ```sudo service ssh restart```

### Configure Uncomplicated Firewall
Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123).
1. ```$ sudo ufw allow 2200/tcp```
2. ```$ sudo ufw allow 80/tcp```
3. ```$ sudo ufw allow 123/udp```
4. ```$ sudo ufw deny 22```$
5. ```$ sudo ufw enable```
_______________________
## Give grader access
Giving the grader access to the application via SSH requires 4 steps:
1. Create a new user account named ```grader```
2. Give ```grader``` permission to ```sudo```
3. Create an SSH key pair for ```grader``` using the ```ssh-keygen``` tool.
4. Log in as new grader user via SSH
5. Disable password authentication


### Create new user
1. To create a new user run the command ```sudo adduser grader```. The setup requires the setup of a password, which is shared with the Udacity team in a private note.
2. To give the new ```grader``` user the permission to ```sudo``` run the command ```$ sudo visudo```. Below the line ```root ALL...``` add a new line ```$grader ALL=(ALL:ALL) ALL```. Save and exit the file.
3. Create a SSH key pair for grader using the ```ssh-keygen``` tool on the **local machine**. In a separate terminal window, run these commands to generate a new key: ```cd ~/.ssh```, ```ssh-keygen -f ~/.ssh/grader_key.rsa```, ```grader```, ```*PASSWORD*``` ```cat ~/.ssh/grader_key.rsa.pub```. The content of this document has to be copied inside the ```authorized_keys``` folder on the Ubuntu server. On the **remote machine** run the command ```touch /home/grader/.ssh/authorized_keys``` and copy+paste the key. To protect the folder from unauthroized edits, change the access permission with the commands: ```sudo chmod 700 /home/grader/.ssh```, ```sudo chmod 644 /home/grader/.ssh/authorized_keys```, ```sudo chown -R grader:grader /home/grader/.ssh```, ```sudo service ssh restart```
4. Log into the remote VM as the new grader user with the command ```ssh -i ~/.ssh/grader_key.rsa -p 2200 grader@18.197.158.41```. The command prompt should change to ```ubuntu@18.197.158.41:~$``` (depending on prompt customizations).
5. To enforce SSH key based auhentication, open the sshd config file with ```sudo nano /etc/ssh/sshd_config``` and edit the line ```PasswordAuthentication``` to no. Restart with ```sudo service ssh restart```.

## Prepare deployment
This stage required 4 steps:
1. Configure the local timezone to UTC.
2. Configure Locale
3. Install and configure Apache to serve a Python mod_wsgi application.
4. Install virtual environment
5. Install necessary packages
6. Install and configure PostgreSQL
7. Install git, Flask

### Configuring Time Zone
To set the timezone to UTC, run the command ```sudo dpkg-reconfigure tzdata``` and choose ```None of the above``` from the initial selection. Then choose ```UTC``` in the new window.

### Configuring Locale Setting
I was running into an error ```unsupported locale setting``` and ran the following commands based on a [stackoverflow](https://stackoverflow.com/questions/14547631/python-locale-error-unsupported-locale-setting) answer by Muhammad Hassan: ```export LC_ALL="en_US.UTF-8"```, ```export LC_CTYPE="en_US.UTF-8"```, ```sudo dpkg-reconfigure locales```

### Install and configure Apache
To install ```apache2``` run the command ```sudo apt-get install apache2```.
When navigating to the public IP of the project, the server should not show a default apache page similar to this one:
![Apache Default Page](images/apache.png)

Due to the upcoming deprecation of Python2 I decided to re-write my project submission for the 2nd project in Python3. This requires a Python3 ```mod_wsgi package``` that's installed as follows:
```sudo apt-get install libapache2-mod-wsgi-py3
sudo a2enmod wsgi
sudo service apache2 start
```


### Install PostgreSQL
To install PostgreSQL run these commands
```
sudo apt-get install libpq-dev python-dev
sudo apt-get install postgresql postgresql-contrib
```

### Install git
Run the follow command: ```sudo apt-get install git```. Configure git with your credentials. If you have no experience with Git, this free [Udacity course](https://eu.udacity.com/course/how-to-use-git-and-github--ud775) is helpful


### Deploy the application
#### Clone Application
Create a new directory for the app and change the access permissions:
```
sudo mkdir /var/www/catalog
sudo chown -R grader:grader catalog
```
Move into the directory and clone the application from Github:
```
$ cd /var/www/catalog
$ git clone https://github.com/nckre/Udacity-FullstackND-Project2-ItemCatalog.git catalog
```
Create a ```catalog.wsgi``` file and paste the following content via ```$ sudo nano /var/www/catalog/catalog.wsgi```:
```
activate_this = '/var/www/catalog/catalog/venv3/bin/activate_this.py'
with open(activate_this) as file_:
    exec(file_.read(), dict(__file__=activate_this))

#!/usr/bin/python3
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/catalog/")
sys.path.insert(1, "/var/www/catalog/")

from catalog import app as application
application.secret_key = 'super_secret_key'
```

#### Update App Code
The app will require several changes to run on a webserver:
1. Change name from ```project.py``` to ```__init__.py```
2. Edit the app to make the following changes (user CTRL+F to find them):
* Change the SQL DB from sqlite to postgresql by replacing ```engine = create_engine('sqlite:///animalcatalog.db',connect_args={'check_same_thread': False}, poolclass=StaticPool, echo=True)``` with ```engine = create_engine('postgresql://catalog:catalog@localhost/catalog')```.
Do the same in the ```database_setup.py``` file where ```engine = create_engine('sqlite:///animalcatalog.db')``` needs to be replaced with ```engine = create_engine('postgresql://catalog:catalog@localhost/catalog')```.
* Remove the 2 lines ```app.debug = True``` and ```app.run(host='0.0.0.0', port=8000)``` and replace them with the single line ```app.run()``` in the ```__init__.py``` file.
3. Change the path for the ```client_secrets.json``` in the app with these 2 edits:
* ```CLIENT_ID = json.loads(open('client_secrets.json', 'r').read())['web']['client_id']``` becomes ```CLIENT_ID = json.loads(open('/var/www/catalog/catalog/client_secrets.json', 'r').read())['web']['client_id']```
* ```oauth_flow = flow_from_clientsecrets('client_secrets.json', scope='')``` becomes ```oauth_flow = flow_from_clientsecrets('/var/www/catalog/catalog/client_secrets.json', scope='')```

#### Install virtual environment
The rest of the application is configured in a virtual environment. This [article](https://www.geeksforgeeks.org/python-virtual-environment/) by geeksforgeeks explains some of the advantages.

To install the virtual environment, run these 2 commands:
```
sudo apt-get install python3-pip
sudo pip install virtualenv
```
Then, switch to the directory and activate it:
```
cd /var/www/catalog/catalog
virtualenv -p python3 venv3
sudo chown -R grader:grader venv3/
. venv3/bin/activate
```

#### Install Packages
Install all the packages that are imported in the python file for Python3 by running these commands:
```
pip3 install httplib2
pip3 install requests
pip3 install --upgrade oauth2client
pip3 install sqlalchemy
sudo apt-get install libpq-dev
pip3 install psycopg2
```

#### Configure Virtual Host
Create a virtual host config file for the catalog and edit it via ```$ sudo nano /etc/apache2/sites-available/catalog.conf.```. Change the content to:
```
<VirtualHost *:80>
    ServerName 52.34.208.247
    ServerAlias ec2-52-34-208-247.us-west-2.compute.amazonaws.com
    ServerAdmin admin@52.34.208.247
    WSGIDaemonProcess catalog python-path=/var/www/catalog:/var/www/catalog/venv/lib/python2.7/site-packages
    WSGIProcessGroup catalog
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
```
To enable the catalog run ```$ sudo a2ensite catalog``` and restart the apache2 with ```$sudo service apache2 reload```.

#### Configure DB
The project also requires to not allow remote connections and create a new database user named ```catalog``` that has limited permissions to the catalog application database. To set this up, run the following commands:
```
sudo su - postgres
psql
CREATE USER catalog WITH PASSWORD 'catalog';
ALTER USER catalog CREATEDB;
CREATE DATABASE catalog WITH OWNER catalog;
\c catalog
REVOKE ALL ON SCHEMA public FROM public;
GRANT ALL ON SCHEMA public TO catalog;
\q
exit
. venv3/bin/activate
python /var/www/catalog/catalog/database_setup.py
```
Then, open the ```pg_hba.conf``` file with ```sudo nano /etc/postgresql/9.5/main/pg_hba.conf``` and make sure the the following permission are set:
```
local   all             postgres                                peer
local   all             all                                     peer
host    all             all             127.0.0.1/32            md5
host    all             all             ::1/128                 md5
```

#### Change Google Authentication
To make the OAUTH work with the webapplication, some last changes are needed to the [Google Developer Console](https://console.developers.google.com). Google does not allow to directly whitelist IP addresses, so either the host name has to be used or a service like [xip.io](http://xip.io).

In the Google console's applications tab "OAuth Consent Screen" the AWS domain has to be whitelisted by adding ```ec2-18-197-158-41.eu-central-1.compute.amazonaws.com``` to the list of authorized domains. Then, in the tab "Credentials" the "Authorized JavaScript origins" have to be modified to allow ```http://ec2-18-197-158-41.eu-central-1.compute.amazonaws.com``` and the "Authorized redirect URIs" need to include:
* ```http://ec2-18-197-158-41.eu-central-1.compute.amazonaws.com/oauth2callback```
* ```http://ec2-18-197-158-41.eu-central-1.compute.amazonaws.com/classes/```



#### Special thanks to [stueken](https://github.com/stueken/FSND-P5_Linux-Server-Configuration) who inspired [iliketomatoes](https://github.com/iliketomatoes/linux_server_configuration) who in turn inspired [kcalata](https://github.com/kcalata/Linux-Server-Configuration). All of their detailed READMEs were very helpful.
