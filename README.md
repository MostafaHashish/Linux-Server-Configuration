# Project 6: Linux Server Configuration Project
Linux Server Configuration project,part of the Udacity [Full Stack Web Developer Nanodegree](https://www.udacity.com/course/full-stack-web-developer-nanodegree--nd004).

## Explanation
Installation of a Linux server and prepare it to host web applications.Secure my server from a number of attack vectors, install and configure a database server, and deploy one of my existing web applications onto it.

## Sever Details
* Public IP : [159.65.162.190](159.65.162.190)
* Server OS : Ubuntu 16.04 x64
* SSH Port: 2200
* SSH password: grader
* Server Provider: [DigitalOcean](https://m.do.co/c/c1a77ba6d4e1)
* Web Apllication: [Item-Catalog-Application](https://github.com/MostafaHashish/Item-Catalog-Application/tree/LinuxProject)


## How to Run Project
### Setting Up the Server

##### 1. Create a new server
* Start a new Ubuntu Linux server on DigitalOcean.
* Use DigitalOcean to access console(login as root)
```sh
~ $  sudo apt-get update # check for updates
~ $ sudo apt-get -u upgrade # checks list of updates and upgrade it
```
##### 2. Initial setup and secure  the server
* Set the Timezone to UTC using the following code
```sh
~ $ sudo timedatectl set-timezone UTC
```
* And use this to verify the status
```sh
~ $ sudo timedatectl status
```
##### 3. Change the SSH Port from 22 to 2200
* Open the SSH configuration
```sh
~ $ sudo nano /etc/ssh/sshd_config
```
* Change Port to **2200**
* Save and restart the ssh
```sh
~ $ sudo service ssh restart
```
##### 4. Configure the Firewall (UFW)
```sh
~ $ sudo ufw status #check status

~ $ sudo ufw default deny incoming #deny ALL incoming
~ $ sudo ufw default allow outgoing #allow ALL outgoing

~ $ sudo ufw allow 2200/tcp #allow SSH
~ $ sudo ufw allow 80/tcp #allow HTTP
~ $ sudo ufw allow 123/udp #allow NTP

~ $ sudo ufw enable # turn on the firewall
~ $ sudo ufw status #check status
```

##### 5. Create a new user
* Create a new user called grader.
```sh
~ $ sudo adduser grader
```
* Give grader the permission to perform sudo commands(without password).
```sh
~ $ sudo visudo #open /etc/sudoers files
#find the following line
root    ALL=(ALL:ALL) ALL
#and add the following line to give the grader sudo permission
grader    ALL=(ALL:ALL) NOPASSWD:ALL
```
* Setup an SSH key for the grader by using the **ssh-keygen**.
```sh
#On Local Machine

type the following code on local machine's terminal to generate ssh key.
~ $ ssh-keygen
change the files path and name to ~/.ssh/grader
read the public key with the following code
~ $ cat ~/.ssh/grader.pub

#Things to do on the remote machine/console
~ $ su - grader #switch to user grader
~ $ nano ~/.ssh/authorized_keys #create file and folder for authorize key.

#Now go back to the local machine and copy the public key and paste it on the  authorized_keys file.

~ $ chmod 700 ~/.ssh && chmod 600 ~/.ssh/* #set the permission for those folder and files
~ $ sudo nano /etc/ssh/sshd_config #let key-based authentication the only way to login
change the line PasswordAuthentication yes to no
remember to save it
```
##### 6. Use SSH login as grader
* Use following code on the local machine to login as grader
```sh
~ $ ssh grader@159.65.162.190 -p 2200 -i ~/.ssh/grader
```
##### 7. Install Apache and mod_wsgi
```sh
~ $ sudo apt-get install apache2
```
* visit  ip number to check if it works. if the default page was shown, it works correctly
* then we could install mod_wsgi(depends on web-app built-n python version)
```sh
~ $ sudo apt-get install libapache2-mod-wsgi #only for python2
```
##### 8. Install and setup PostgreSQL
* psql is our database server, so we need to install it first
```sh
~ $ sudo apt-get install postgresql
```
* psql is default disable the remote connections, but we still need to make sure it by visiting it's configure file
```sh
~ $ sudo nano /etc/postgresql/9.5/main/pg_hba.conf
```
* we need to make sure that it only allows connections from localhost (either 127.0.0.1 for IPv4 or ::1 for IPv6) and no external connections.
* after making sure everything works correctly we will need to setup the psql for further usage.
```sh
~ $ sudo -u postgres psql #use this code to get into psql prompt
then create a database with username and password for security

CREATE ROLE catalog WITH LOGIN PASSWORD 'topsecret'; # catalog is user and topsecret is password

then create a new database called catalogdb

CREATE DATABASE catalogdb;

use \q to exit psql prompt
```
##### 8. adjust Catalog Application
* in ubuntu it might not work ,so we will need to change it.
```sh
Change 'client_secrets.json'  to static location
'/var/www/itemcatalog/app/client_secrets.json'
```
* Add the **ip address** to the **Authorized JavaScript origins** of **Google’s Developer Console**.

##### 9. Deploy the Item Catalog Application
* clone **LinuxProject branch** from **GitHub** to the correct folder
```sh
a. Go to the correct directory
~ $ cd /var/www/itemcatalog/

b. clone the correct branch to the correct folder
~ $ git clone --single-branch https://github.com/MostafaHashish/Item-Catalog-Application --branch LinuxProject app

now the application is located inside  /var/www/itemcatalog/app
```
* Set up the virtual machine
```sh
a. install pip in order to install virtualenv
~ $ sudo apt-get install python-pip

b. use pip to install virtualenv
~ $ sudo pip install virtualenv

c. make sure we are on the right folder
~ $ cd /var/www/itemcatalog/app/
NOTE :~ $  sudo chmod o+w /var/www/itemcatalog/app/  then
      ~ $  cd /var/www/itemcatalog/app/
       if it does not work.

d. initiate virtual environment and select the correct python version
~ $ virtualenv -p python venv

e. activate the virtual environment
~ $ source venv/bin/activate

f. install the requirements python packages
~ $ pip install -r requirements.txt

g. initiate our database
~ $ python lotsofitems.py
~ $ python project.py #after intial the server use ctrl+C to get out

h. Deactivate our virtual enviroment
~ $ deactivate
```
* now we will need to create the wsgi file
```sh
~ $ sudo nano /var/www/itemcatalog/app.wsgi
```
* and we will need to add the following content inside
```sh
import sys
sys.path.insert(0, '/var/www/itemcatalog/app')
#activate_this is for activate the packages for virtual environment
activate_this = '/var/www/itemcatalog/app/venv/bin/activate_this.py'
execfile(activate_this, dict(__file__=activate_this))
from project import app as application
```

* create a new configuration file for our applications virtual host by running
```sh
~ $ sudo nano /etc/apache2/sites-available/app.conf
```
* and add the following content inside

```sh
<VirtualHost *:80>
  ServerName ubuntu-s-1vcpu-1gb-nyc3-01
  ServerAlias  159.65.162.190
  ServerAdmin local@local
  WSGIScriptAlias / /var/www/itemcatalog/app.wsgi
  <Directory /var/www/itemcatalog/app/>
      Order allow,deny
      Allow from all
  </Directory>
 Alias /static /var/www/itemcatalog/app/static
  <Directory /var/www/itemcatalog/app/static/>
      Order allow,deny
      Allow from all
  </Directory>
  ErrorLog ${APACHE_LOG_DIR}/error.log
  CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
* Enable our virtual host.
```sh
~ $ sudo a2ensite itemcatalog
```
* Restart Apache
```sh
~ $ sudo apache2ctl restart
```
##### 10. Finish
* try [159.65.162.190](159.65.162.190)  to test the application.
## References:
https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps
