# Linux Web Server Configuration
after we build the project now we are hosting it, on a real live linux server and this doc will quide you to configure your server and secure it.

##Access The server App
 - ip: 104.197.229.70
 - ssh port : 2200
 - user : grader


# Software Installed
The Following software :

    fail2ban
    ntp
    apache2
    libapache2-mod-wsgi
    postgresql
    postgresql-contrib
    git
    python-flask
    python-flask-sqlalchemy
    python-psycopg2
    python-oauth2client

###   setup the server :

ssh into the server that you have either google cloud or AWS

first step is to create a RSA public key (never do this on the server!):

create and .ssh folder :



    mkdir .ssh


give it prommesion of 600:



    sudo chmod 600 .ssh



now lets create the key:



    ssh-keygen


it will prompt to provide the location on the RSA key
/home/grader/.ssh/~yourkeyname~
then the keyphrase

give it a premmsion 400 if the you got a error that its too open



    sudo chmod 400 /.ssh/~key~


very important note ~

if you using google cloud you should add it in the web control panel

to able to access the ssh

got to >compute engine > vm instance > instance-name > edit > ssh keys

paste the Rsa.pub content in the box

and you are good to go


###  SSH
the RSA key we created it befor



    ssh root@104.197.229.70 -i ~/.ssh/nemo




###  Create new user

once logged in create a new user with name grader



    adduser grader


dont worry about the password you can set it what you want

###  Grant sudo access
create a user instance
and edit it :



    touch /etc/sudoers.d/grader
    nano /etc/sudoers.d/grader



type inside the following :




    grader ALL=(ALL) NOPASSWD:ALL



this line will give access to sudo to the user


###   Configure SSH for New User
note : if you using google cloud skip this if you done the previos step

other lets continue :
Next, add the ssh authorized key for the new user. Temporarily log in as the new user:



    sudo -su grader



Create the .ssh directory and the authorized_keys file:


    cd /home/grader

    mkdir .ssh

    touch .ssh/authorized_keys

    nano .ssh/authorized_keys



and paste the publickey.pub you will find it in the .ssh folder


Set the ssh file permissions:



    chmod 700 .ssh
    chmod 644 .ssh/authorized_keys




test the connection :



    ssh grader@104.197.229.70 -i ~/.ssh/nemo.rsa




###  Update Packages :

type the following command



    sudo apt-get update
    sudo apt-get upgrade



let it run

Now to allow security updats install the package


    sudo apt-get install unattended-upgrades


and enable it :



    sudo dpkg-reconfigure --priority-low unattended-upgrades



###   SSH Config :
to change the ssh default port from 22 to 2200
edit the following file :



    sudo nano /etc/ssh/sshd_config



change it to the following


    Port 2200



###  Remove Root Login :
change the following to no:


    #Authentication:
    PermitRootLogin no



###  force ssh login :
change the following to:



    # Change to no to disable tunnelled clear text passwords
    PasswordAuthentication no



save and edit

now restart the service:



    sudo service ssh restart

'

NOTE: please be carefull with this and conform that you still can accees the user account


###  Configure Firewall :

Configure (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123):



    sudo ufw default deny incoming

    sudo ufw allow 2200

    sudo ufw allow 80

    sudo ufw allow 123

    sudo ufw enable




NOTE: this is not enough if you using google cloud you should do the following :



    gcloud compute firewall-rules create ssh --allow tcp:2200





    gcloud compute firewall-rules create www --allow tcp:80





    gcloud compute firewall-rules create ntp --allow tcp:123



still you must enable the http method in the control panel of your vm


for more information please the the refrencess section bellow



###   Firewall Monitoring

block IP addresses that fail to correctly log in



    sudo apt-get install fail2ban



next :



    sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
    sudo nano /etc/fail2ban/jail.local



update it as the following :




    [ssh]

    enabled  = true
    banaction = ufw-ssh
    port     = 2200
    filter   = sshd
    logpath  = /var/log/auth.log
    maxretry = 3



create the ufw ssh action referenced:



    sudo touch /etc/fail2ban/action.d/ufw-ssh.conf
    sudo nano /etc/fail2ban/action.d/ufw-ssh.conf



define it as following :



    [Definition]
    actionstart =
    actionstop =
    actioncheck =
    actionban = ufw insert 1 deny from <ip> to any app OpenSSH
    actionunban = ufw delete deny from <ip> to any app OpenSSH


finally :




    sudo service fail2ban restart




###  Set Local Timezone :

run the following command and chooce UTC


    sudo dpkg-reconfigure tzdata



###  Install Apache :

installing apache



 sudo apt-get install apache2



###  install Install mod_wsgi :

run :



    sudo apt-get install libapache2-mod-wsgi python-dev



enable it :



    sudo a2enmod wsgi



start or restart apache :



    sudo service apache2 start




###   Deploing the APP :

Use the following command to move to the /var/www directory:



    cd /var/www





    mkdir flaskApp
    cd flaskApp



git clone the app :



    git clone https://github.com/deyanemo/Item-Catalog.git flaskApp





Now, create the __init__.py file that will contain the flask application logic.




    sudo nano __init__.py



Add following logic to the file:



    from flask import Flask
    app = Flask(__name__)
    @app.route("/")
    def hello():
        return "Hello, I love Digital Ocean!"
    if __name__ == "__main__":
        app.run()



Save and close the file.


###  Install Flask

We will use pip to install virtualenv and Flask. If pip is not installed, install it on Ubuntu through apt-get.



    sudo apt-get install python-pip
    sudo pip install virtualenv





    sudo virtualenv venv



Now, install Flask in that environment by activating the virtual environment with the following command:


    source venv/bin/activate


### install Flask



    sudo pip install Flask



change the app name :



    sudo python __init__.py



To deactivate the environment



    deactivate



###  New Virtual Host
run the following commands to configure the apache sites



    sudo nano /etc/apache2/sites-available/FlaskApp.conf



paste inside of it the following after changing your apps prefixes :



    <VirtualHost *:80>
            ServerName mywebsite.com
            ServerAdmin admin@mywebsite.com
            WSGIScriptAlias / /var/www/FlaskApp/flaskapp.wsgi
            <Directory /var/www/FlaskApp/FlaskApp/>
                Order allow,deny
                Allow from all
            </Directory>
            Alias /static /var/www/FlaskApp/FlaskApp/static
            <Directory /var/www/FlaskApp/FlaskApp/static/>
                Order allow,deny
                Allow from all
            </Directory>
            ErrorLog ${APACHE_LOG_DIR}/error.log
            LogLevel warn
            CustomLog ${APACHE_LOG_DIR}/access.log combined
    </VirtualHost>



Save and close the file.

now we will enable the virtual host :



    sudo a2ensite FlaskApp


###  Create the .wsgi File

Apache uses the .wsgi file to serve the Flask app. Move to the /var/www/FlaskApp



    cd /var/www/FlaskApp
    sudo nano flaskapp.wsgi


paste inside it :



    #!/usr/bin/python
    import sys
    import logging
    logging.basicConfig(stream=sys.stderr)
    sys.path.insert(0,"/var/www/FlaskApp/")

    from FlaskApp import app as application
    application.secret_key = 'Add your secret key'



    at last retart apache to changes takes effects :




    sudo service apache2 restart




###  refrencess :

https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps
https://cloud.google.com/compute/docs/
