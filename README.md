# Linux-Server-Configuration

This page explains how to secure and set up a Linux distribution on a virtual machine, install and configure a web and database server to host a web application.

```
  iP address: 54.210.11.70

  Accessible SSH port: 2200

  Application URL: http://ec2-54-210-11-70.compute-1.amazonaws.com/
```

## Step

### 1. Launch Virtual Machine and SSH into the server
- From the `Account` menu on Amazon Lightsail, click on `SSH keys` tab and download the Default Private Key.
- Move this private key file named `LightsailDefaultPrivateKey*.pem` into the local folder `~/.ssh` and rename it `lightsail_key.rsa`.
- Change the key permission so that only owner can read and write:
  In the terminal, type: `chmod 600 ~/.ssh/lightsail_key.rsa`.
- SSH into the instance:: `ssh -i ~/.ssh/lightsail_key.rsa ubuntu@54.210.11.70`

### 2. User Management: Create a new user and give user the permission to sudo
  * Add User grader
  
  ```
    $ sudo adduser grader
  ```
  * Give Sudo Access to grader
  
  ```
    $ sudo nano /etc/sudoers.d/grader
  ```
  * Add following line to this file
  
  ```
    grader ALL=(ALL:ALL) ALL
  ```
  
### 3. Set-up SSH keys for user grader
  *  Generate an encryption key on your local machine
  
   3.1 Go to the directory where you want to save the Key, and run the following command:
   ```
    $ ssh-keygen -t rsa
   ```
      followed by the name of the key. By default, the keys will be stored in the ~/.ssh directory within your user's 
      home directory.
      
   3.2. Place the public key on the server that we want to use:
   ```
    $ ssh-copy-id grader@54.210.11.70  -i grader_key.pub
   ```
   3.3. Log into remote VM as root user and :
   
   ```
  mkdir /home/grader/.ssh
  chown grader:grader /home/grader/.ssh
  chmod 700 /home/grader/.ssh
  cp /root/.ssh/authorized_keys /home/grader/.ssh/
  chown grader:grader /home/grader/.ssh/authorized_keys
  chmod 644 /home/grader/.ssh/authorized_keys
   ```
  
  Now we can log into the remote VM through ssh with the following command: 
  
  ```
   $ ssh -i lightsail_key.rsa grader@54.210.11.70 
  ```

### 4. Change SSH port from 22 to 2200
  * Find the **Port line** in the same file above, i.e */etc/ssh/sshd_config* and edit it to 2200.
  * Run `sudo service ssh restart` to restart the service.
  
### 5. Disable root login
  * Run `$ sudo nano /etc/ssh/sshd_config`.
  * Find the **PasswordAuthentication** line and edit it to no.
  * Change `PermitRootLogin without-password` to `PermitRootLogin no`.
  * Save the file.
  * Run `sudo service ssh restart` to restart the service.

  Now need to use the following command to login to the server: `ssh -i ~/.ssh/lightsail_key.rsa grader@54.210.11.70 -p 2200`

### 6. Configure local timezone to UTC
  * Change the timezone to UTC using following command: `$ sudo timedatectl set-timezone UTC`.
  * You can also open time configuration dialog and set it to UTC with: `$ sudo dpkg-reconfigure tzdata`.

### 7. Update all currently installed packages
  * `sudo apt-get update`.
  * `sudo apt-get upgrade`.

### 8. Configure the Uncomplicated Firewall (UFW)
  ```
   $ sudo ufw default deny incoming
   $ sudo ufw default allow outgoing
   $ sudo ufw allow 2200/tcp
   $ sudo ufw allow www
   $ sudo ufw allow ntp
   $ sudo ufw enable
  ```

### 9. Install Apache to serve a Python mod_wsgi application
 ```
  $ sudo apt-get install apache2 libapache2-mod-wsgi
 ```
 * Enable mod_wsgi:
 
 ```
  $ sudo a2enmod wsgi
 ```

### 10. Install and configure PostgreSQL
  * Installing PostgreSQL Python dependencies:
  - While logged in as `grader`, install PostgreSQL:
 `sudo apt-get install postgresql`.

  - Switch to the `postgres` user: `sudo su - postgres`.
  - Open PostgreSQL interactive terminal with `psql`.
  - Create the `catalog` user with a password and give them the ability to create databases:
  ```
  postgres=# CREATE ROLE catalog WITH LOGIN PASSWORD 'catalog';
  postgres=# ALTER ROLE catalog CREATEDB;
  ```

  - Exit psql: `\q`.
  - Switch back to the `grader` user: `exit`.
  - Create a new Linux user called `catalog`: `sudo adduser catalog`. Enter password and fill out information.
  - Give to `catalog` user the permission to sudo. Run: `sudo visudo`.
  - Search for the lines that looks like this:
  ```
  root    ALL=(ALL:ALL) ALL
  grader  ALL=(ALL:ALL) ALL
  ```

  - Below this line, add a new line to give sudo privileges to `catalog` user.
  ```
  root    ALL=(ALL:ALL) ALL
  grader  ALL=(ALL:ALL) ALL
  catalog  ALL=(ALL:ALL) ALL
  ```

  - Save and exit using CTRL+X and confirm with Y.

 ### 11. Install git
  - logged in as `grader`, install `git`: `sudo apt-get install git`.

 ### 12. Deploy Flask Application:

 #### step-1 
 *Make a catalog.wsgi file to serve the application over the mod_wsgi. with content:

    ```
     $ touch catalog.wsgi && nano catalog.wsgi
    ```

    ```
    import sys
    import logging
    logging.basicConfig(stream=sys.stderr)
    sys.path.insert(0, "/var/www/catalog/")

    from catalog import app as application
    ```
#### step-2 
*Make a catalog.wsgi file to serve the application over the mod_wsgi. with content:  
* Make a *catalog* named directory in */var/www*

    ```
      $ sudo mkdir /var/www/catalog
    ```

  * Change the owner of the directory *catalog*

    ```
     $ sudo chown -R grader:grader /var/www/catalog
    ```
#### step-3
* Now create a virtual environment for our flask application. use pip to install virtualenv and Flask. Install pip :
    `sudo apt-get install python-pip`
* Install virtualenv:   `sudo pip install virtualenv`
* Set enviornment name using : `sudo virtualenv venv`
* Install Flask in that environment by activating the virtual environment using : `source venv/bin/activate`
* Install Flask using : `sudo pip install Flask`
* Install the following dependencies:
  ```
  pip install httplib2
  pip install requests
  pip install --upgrade oauth2client
  pip install sqlalchemy
  pip install flask
  sudo apt-get install libpq-dev
  pip install psycopg2
  ```
* To deactivate the environment : `deactivate`

#### step-4
* Run - `sudo nano /etc/apache2/sites-available/catalog.conf`
* configure the virtual host adding your Servername:
```
  <VirtualHost *:80>
    ServerName 54.210.11.70
    ServerAlias ec2-54-210-11-70.compute-1.amazonaws.com
    ServerAdmin grader@54.210.11.70
    WSGIDaemonProcess catalog python-path=/var/www/catalog/:/var/www/catalog/venv/lib/python2.7/site-packages
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
 * Enable virtual host using :  `sudo a2ensite catalog`

### 13. Clone the Catalog app from Github

    ```
      $ cd /var/www/catalog

      $ git clone https://github.com/ekarfan/OAuth2.0.git catalog
    
    ```

  * Rename the `application.py` file to `__init__.py` using: `mv application.py __init__.py`.
    - In `__init__.py`:
    ```
    # app.run(host="0.0.0.0", port=5000, debug=True)
    app.run()
    ```
    - In `database_setup.py`, `database_init.py`, `__init__.py`:
    ```
    # engine = create_engine("sqlite:///catalog.db")
    engine = create_engine('postgresql://catalog:catalog@localhost/catalog')
    ``` 
    * Run the database_setup.py and database_init.py once to setup database with dummy data:
  
    ```
    $ python database_setup.py
    $ python database_init.py
    ```

### 14. Restart Apache to launch the app

   ```
    $ sudo service apache2 restart
   ```


