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
  
#### 3. Set-up SSH keys for user grader
  *  Generate an encryption key on your local machine
  
   i. Go to the directory where you want to save the Key, and run the following command:
   
   ```
    $ ssh-keygen -t rsa
   ```
      followed by the name of the key. By default, the keys will be stored in the ~/.ssh directory within your user's 
      home directory.
      
   ii. Place the public key on the server that we want to use:
   
   ```
    $ ssh-copy-id grader@54.210.11  -i grader_key.pub
   ```
   iii. Log into remote VM as root user and open the following file:
   
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

#### 4. Change SSH port from 22 to 2200
  * Find the **Port line** in the same file above, i.e */etc/ssh/sshd_config* and edit it to 2200.
  * Run `sudo service ssh restart` to restart the service.
  
#### 5. Disable root login
  * Run `$ sudo nano /etc/ssh/sshd_config`.
  * Find the **PasswordAuthentication** line and edit it to no.
  * Change `PermitRootLogin without-password` to `PermitRootLogin no`.
  * Save the file.
  * Run `sudo service ssh restart` to restart the service.

  Now need to use the following command to login to the server: `ssh -i ~/.ssh/lightsail_key.rsa grader@54.210.11.70 -p 2200`

#### 6. Configure local timezone to UTC
  * Change the timezone to UTC using following command: `$ sudo timedatectl set-timezone UTC`.
  * You can also open time configuration dialog and set it to UTC with: `$ sudo dpkg-reconfigure tzdata`.

#### 7. Update all currently installed packages
  * `sudo apt-get update`.
  * `sudo apt-get upgrade`.

#### 8. Configure the Uncomplicated Firewall (UFW)
  ```
   $ sudo ufw default deny incoming
   $ sudo ufw default allow outgoing
   $ sudo ufw allow 2200/tcp
   $ sudo ufw allow www
   $ sudo ufw allow ntp
   $ sudo ufw enable
  ```

#### 9. Install Apache to serve a Python mod_wsgi application
 ```
  $ sudo apt-get install apache2 libapache2-mod-wsgi
 ```
 * Enable mod_wsgi:
 
 ```
  $ sudo a2enmod wsgi
 ```

#### 10. Install and configure PostgreSQL
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

 #### 11. Install git
  - While logged in as `grader`, install `git`: `sudo apt-get install git`.

 #### 12. Deploy Flask Application:

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
 Save and close the file.
 * Enable virtual host using :  `udo a2ensite catalog`

#### 15. Clone the Catalog app from Github

  * Make a *catalog* named directory in */var/www*

    ```
      $ sudo mkdir /var/www/catalog
    ```

  * Change the owner of the directory *catalog*

    ```
     $ sudo chown -R grader:grader /var/www/catalog
    ```

  * Clone the **bookCatalog** to the catalog directory:

    ```
     $ git clone https://github.com/sagarchoudhary96/P5-Item-Catalog.git catalog
    ```

  * Change the branch of repo **bookCatalog**  to **deployment**:

    ```
     $ cd catalog && git checkout deployment
    ```

  * 
  * Rename the `application.py` file to `__init__.py` using: `mv application.py __init__.py`.
    - In `__init__.py`:
    ```
    # app.run(host="0.0.0.0", port=8000, debug=True)
    app.run()
    ```
    - In `database_setup.py`, `database_init.py`, `__init__.py`:
    ```
    # engine = create_engine("sqlite:///catalog.db")
    engine = create_engine('postgresql://catalog:PASSWORD@localhost/catalog')
    ``` 
    * Run the database_setup.py and database_init.py once to setup database with dummy data:
  
    ```
    $ python database_setup.py
    $ python dummybooks.py
    ```

  #### 16. Edit the default Virtual File with following content:

  ```
    $  sudo nano /etc/apache2/sites-available/catalog.conf
  ```


  ```
  <VirtualHost *:80>
    ServerName XX.XX.XX.XX
    ServerAdmin sagar.choudhary96@gmail.com
    WSGIScriptAlias / /var/www/catalog/bookCatalog.wsgi
    <Directory /var/www/catalog/>
        Order allow,deny
        Allow from all
    </Directory>
    Alias /static /var/www/catalog/static
    <Directory /var/www/catalog/static/>
        Order allow,deny
        Allow from all
    </Directory>
  </VirtualHost>
  ```

### Step 14.1: Install the virtual environment and dependencies

- While logged in as `grader`, install pip: `sudo apt-get install python2-pip`.
- Install the virtual environment: `sudo apt-get install python-virtualenv`
- Change to the `/var/www/catalog/catalog/` directory.
- Create the virtual environment: `sudo virtualenv -p python3 venv3`.
- Change the ownership to `grader` with: `sudo chown -R grader:grader venv3/`.
- Activate the new environment: `. venv3/bin/activate`.
- Install the following dependencies:
  ```
  pip install httplib2
  pip install requests
  pip install --upgrade oauth2client
  pip install sqlalchemy
  pip install flask
  sudo apt-get install libpq-dev
  pip install psycopg2
  ```

- Run `python3 __init__.py` and you should see:
  ```
  * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
  ```

- Deactivate the virtual environment: `deactivate`.
#### 17. Restart Apache to launch the app

   ```
    $ sudo service apache2 restart
   ```





### 3 - Update and upgrade all currently installed packages
    
1. Update the list of available packages and their versions:  
  `$ sudo apt-get update`
2. Install newer vesions of packages you have:  
  `$ sudo sudo apt-get upgrade`

#### 5** - Include cron scripts to automatically manage package updates

1. Install the unattended-upgrades package:  
  `$ sudo apt-get install unattended-upgrades`
2. Enable the unattended-upgrades package:  
  `$ sudo dpkg-reconfigure -plow unattended-upgrades`

### 6 - Change the SSH port from 22 to 2200 and configure SSH access
Source: [Ask Ubuntu][8]  

1. Change ssh config file:
  1. Open the config file:  
    `$ vim /etc/ssh/sshd_config` 
  2. Change to Port 2200.
  3. Change `PermitRootLogin` from `without-password` to `no`.
  4. * To get more detailed logging messasges, open `/var/log/auth.log` and change LogLevel from `INFO` to `VERBOSE`. 
  5. Temporalily change `PasswordAuthentication` from `no` to `yes`.
  6. Append `UseDNS no`.
  7. Append `AllowUsers NEWUSER`.  
**Note:** All options on [UNIXhelp][9]
2. Restart SSH Service:  
  `$ /etc/init.d/ssh restart` or `# service sshd restart` 
3. Create SSH Keys:  
  Source: [DigitalOcean][10]  

  1. Generate a SSH key pair on the local machine:  
    `$ ssh-keygen`
  2. Copy the public id to the server:  
    `$ ssh-copy-id username@remote_host -p**_PORTNUMBER_**`
  3. Login with the new user:  
    `$ ssh -v grader@PUBLIC-IP-ADDRESS -p2200`
  4. Open SSHD config:  
    `$ sudo vim /etc/ssh/sshd_config`
  5. Change `PasswordAuthentication` back from `yes` to `no`.
4. *Get rid of the warning message `sudo: unable to resolve host ...` when sudo is executed:  
Source: [Ask Ubuntu][11]  

  1. Open `$ vim /etc/hostname`.
  2. Copy the hostname.
  3. Append the hostname to the first line:  
    `$ sudo sudonano /etc/hosts`
5. * Simplify SSH login:  
  1. Logout of the SSH instance:  
    `$ exit`
  2. Open the SSH config file on your local machine:  
    `$ sudo vim .ssh/config`
  3. Add the following lines:  
    ```
    Host NEWHOSTNAME
      HostName PUPLIC-IP-ADDRESS
      Port 2200
      User NEWUSER
    ```
  4. Now, you can login into the server more quickly:  
    `$ ssh NEWHOSTNAME`
6. *Handle the message `System restart required` after login:  
Source: [Super User][12]  

  1. List all packages which cause the reboot:  
    `$ cat /var/run/reboot-required.pkgs`
  2. List everything with high security issues:  
    `$ xargs aptitude changelog < /var/run/reboot-required.pkgs | grep urgency=high`
  3. If wanted or needed, reboot the system:  
    `$ sudo shutdown -r now`  
    **Note**: More info on rebooting on [Ask Ubuntu][13].

### 7 - Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)
Source: [Ubuntu documentation][14]  

1. Turn UFW on with the default set of rules:  
  `$ sudo ufw enable` 
2. *Check the status of UFW:  
  `$ sudo ufw status verbose`
3. Allow incoming TCP packets on port 2200 (SSH):  
  `$ sudo ufw allow 2200/tcp` 
4. Allow incoming TCP packets on port 80 (HTTP):  
  `$ sudo ufw allow 80/tcp` 
5. Allow incoming UDP packets on port 123 (NTP):  
  `$ sudo ufw allow 123/udp`  

#### 7** - Configure Firewall to monitor for repeated unsuccessful login attempts and ban attackers
Source: [DigitalOcean][15]  

1. Install Fail2ban:  
  `$ sudo apt-get install fail2ban`
2. Copy the default config file:  
  `$ sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local`
3. Check and change the default parameters:  
    1. Open the local config file:  
      `$ sudo vim /etc/fail2ban/jail.local`
    2. Set the following Parameters:  
    ```  
      set bantime  = 1800  
      destemail = YOURNAME@DOMAIN  
      action = %(action_mwl)s  
      under [ssh] change port = 2220  
    ```  
    
  **Note:** In the next three steps *iptables* is installed. However, the before installed UFW [is actually a frontend for iptables](https://wiki.ubuntu.com/UncomplicatedFirewall) and is set up already. So configuring *iptables* separately (as I did by just following the guide at DigitalOcean) would be a redundant step. So just install *sendmail* and go on with step 7.
