# Linux Web Server Configuration

This README outlines the steps I followed to setup and configure a Linux web server VM according to instructions provided for Project 5: Linux Server Configuration of the Full-Stack Web Developer Nanodegree program. The application hosted on the VM can be found [here](http://52.88.25.245).

## First things First: Logging In
To log into the virtual machine, open a terminal window and type the following:

`ssh -i ~/.ssh/udacity_key.rsa root@52.88.25.245 -p 2200`

Ensure that the file `udacity_key.rsa` is in the `.ssh` folder located in your home directory.

## Step-by-Step Configuration

### Step 1: Add 'grader' user
The first project requirement is to add a user named `grader`, and give that user permission use `sudo`. The instructions don't specify whether the user account should have a password, so the account was created without one:

`sudo adduser grader sudo --disable-password`

Udacity's Linux server configuration course suggested editing the `/etc/sudoers` file by hand to grant `sudo` permission to the `grader` user, but the above command line satisfies that requirement as well. I found that when I tried to use `sudo` with the `grader` account I was prompted for a password, even though I specified that the user account had no password. To solve this problem, I open the `sudoers` file with:

`sudo visudo`

And changed the line:

`%sudo  ALL=(ALL:ALL) ALL`

To:

`%sudo  ALL=(ALL:ALL) NOPASSWD: ALL`.

### Step 2: Update all currently-installed packages
This one is easy, it's just two command lines:

`sudo apt-get update`

`sudo apt-get upgrade`

Important to note: `apt-get update` only updates package lists with information about the newest available versions of installed packages and their dependencies; `apt-get upgrade` actually downloads and installs them. There is no way to combine these commands into a single step.

### Step 3: Change SSH port to 2200
To change the port the SSH service listens on, open the configuration file with an editor:

`sudo nano /etc/ssh/sshd_config`

And change the following line:

`Port 22`

To:

`Port 2200`

Once you're finished, make sure you restart the SSH service (this is critical!):

`sudo service ssh restart`

Restarting the service will reload the configuration from the `sshd_config` file. Without this step, the SSH service will continue listening on port 22. If you're not careful with this step (and the next), you could end up locking yourself out of your VM.

### Step 4: Configuring and enabling a firewall (UFW)
True to its name, Uncomplicated Firewall (UFW) (installed by default on Ubuntu distros) makes fulfilling this requirement pretty simple. The first (and most delicate) step is to allow traffic on port 2200 (recall this is the port the SSH service is configured to listen on):

`sudo ufw allow 2200/tcp`

Also important is, by default, to deny all incoming traffic (unless the configuration specifies otherwise):

`sudo ufw default deny incoming`

And since we're using VM as a web server, it's also pretty important to allow traffic on port 80:

`sudo ufw allow www`

An additional requirement was to allow NTP traffic on port 123:

`sudo ufw allow ntp`

Since the firewall is disabled by default, the firewall must be enabled after making changes to the configuration:

`sudo ufw enable`

### Step 5: Change timezone to UTC
Use this command line:

`dpkg-reconfigure tzdata`

In the GUI, select "None of the above" for your Geographic area, and then "UTC" for your timezone.

### Step 6: Install Apache and configure to serve Python WSGI Applications
Use aptitude to install Apache:

`sudo apt-get install apache2`

To enable Apache to serve Python WSGI applications, only one additional package is needed:

`sudo apt-get install libapache2-mod-wsgi`

### Step 7: Install PostgreSQL and create 'catalog' user
Installing PostgreSQL is simple:

`sudo apt-get install postgresql`

One requirement is to disable remote access to the PostgreSQL server, but this is the case by default, so no additional configuration changes are needed. To create a user for the catalog application, use the `createuser` command line as the user `postgres`:

`sudo -u postgres createuser -S -D -R catalog`

Once the user is created we can connect to the database server:

`sudo -u postgres psql`

Create a `catalog` database on the server:

`CREATE DATABASE catalog;`

And grant the `catalog` user liberal privileges on that database:

`GRANT ALL PRVILEGES ON catalog to catalog;`

Type `\q` to quit the psql command prompt.

### Step 8: Install Git and clone Catalog App project
Installing Git is easy:

`sudo apt-get install git`

To clone a Git repository, use the `clone` command along with the URL of the GitHub repository you wish to clone. The command line to clone my Catalog App repository is:

`git clone http://www.github.com/foahchon/catalog`

Git will create a folder called 'catalog' in your current working directory containing all of the files in the repository. I decided the folder containing my files would go in the `/var/www/` folder.


### Step 9: Modifying Catalog App to run in a WSGI/PostgreSQL environment
This is probably the trickiest part of all; there are a lot of changes that need to be made to the original project to run in the VM's environment. The original project required the use of SQLite database with a filename of `catalog.db`, so first step I took was to all locations where a SQLite connection string was being used:

`sqlite:///catalog.db`

And replace it with the appropriate PostgreSQL connection string:

`postgres://catalog@localhost:5432/catalog`

The next step was to create a WSGI file, `application.wsgi` that the Apache WSGI module could load and run:

    import sys
    import os
    
    sys.path.insert(0, '/var/www/catalog')
    os.chdir('/var/www/catalog`)
    
    from application import app as application

The only requirement for a WSGI module to run is for an object named `application` to be defined. The `sys` and `os` calls are necessary, because without them, Apache is unable to find those modules, even if they are located in the same directory as `application.wsgi`.

The last modification to the application's code was in `application.py`. In the original file, several important properies are set on the `app ` object, but only if `application.py` is the main module, including the app's `secret_key` (which is necessary to run) and the app's `debug` flag to True (which enables the app to write helpful debugging information to Apache's log files). To solve this problem, I simply moved those lines of code to line 26 of `application.py` where the `app` object is first instantiated.

The final step needed to get the application up and running on the web is to edit Apache's "available sites" configuration files:

`sudo nano /etc/apache2/sites-available/000-default.conf`

And add the following line:

`WSGIScriptAlias / /var/www/catalog/application.wsgi`

This line will point all `GET /` requests coming into the server to the `application.wsgi` file we created earlier.

### Step 10: Limit unsuccessful SSH login attempts (extra credit)
UFW makes this requirement pretty simple. If the SSH service were running on its default port of 22, we could simply use the command `sudo ufw limit ssh`, but since we changed the SSH port to 2200, we will instead use the following:

`sudo ufw limit 2200/tcp`

This will deny connections from IP addresses who've attempted to make six or more connections in the last thirty seconds.

### Step 11: Create a cronjob to automatically install software updates (extra credit)
Creating cronjobs is pretty easy on Ubuntu with the use of Crontab. To add new a cronjob, use the following command line:

`sudo crontab -e`

This will open the Crontab configuration file. I appended these lines to the bottom:

    0 7 * * * apt-get update
    1 7 * * * apt-get upgrade

These lines will cause the `apt-get update` command to be run at every day at 3:00AM EST (7:00AM UTC) and `apt-get upgrade` to be run a minute later at 3:01AM EST (7:01AM UTC).

### Step 12: Install monitoring applications with automated availability feedback (extra credit)
To fulfill this requirement, I use logwatch. To install:

`sudo apt-get logwatch`

LogWatch analyzes log files and summarizes them in a report, which you can then e-mail to an address of your choosing. It's possible to configure LogWatch to send an automated e-mail report daily, but I decided to use a cronjob for this purpose instead, since I felt it offered more fine-grained control over the frequency of the reports and other configuration options, in case I wanted to make a change later on. I added this line to my Crontab configuration file:

    0 4 * * * logwatch --detail-level Low --mailto <email hidden> --service http --range today

This will cause a report detailing HTTP activity to e-mailed at 12:00AM EST (5:00AM UTC) each night.

# Thanks
Thanks for checking out my project. Hope you enjoyed!

# Resources
- [Ubuntu Help: Allowing other users to run sudo](https://help.ubuntu.com/community/RootSudo#Allowing_other_users_to_run_sudo)
- [Liquid Web Knowledge Base: Changing The SSH Port](http://www.liquidweb.com/kb/changing-the-ssh-port/)
- [Digital Ocean: How To Setup a Firewall with UFW on an Ubuntu and Debian Cloud Server](https://www.digitalocean.com/community/tutorials/how-to-setup-a-firewall-with-ufw-on-an-ubuntu-and-debian-cloud-server)
- [Slaptijack: Limiting SSH Connections with ufw](http://slaptijack.com/system-administration/limiting-ssh-connections-with-ufw/)
- [Ubuntu Help: Ubuntu Time Management](https://help.ubuntu.com/community/UbuntuTime)
- [Ubuntu Packages Search](http://packages.ubuntu.com/)
- [PostgreSQL Documentation: createuser](http://www.postgresql.org/docs/9.2/static/app-createuser.html)
- [PostgreSQL Documentation: GRANT](http://www.postgresql.org/docs/9.0/static/sql-grant.html)
- [Flask Documentation: mod_wsgi (Apache)](http://flask.pocoo.org/docs/0.10/deploying/mod_wsgi/)
- [Digital Ocean: How To Set Up Apache Virtual Hosts on Ubuntu 14.04 LTS](https://www.digitalocean.com/community/tutorials/how-to-set-up-apache-virtual-hosts-on-ubuntu-14-04-lts)
- [Digital Ocean: How To Install and Use Logwatch Log Analyzer and Reporter on a VPS](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-logwatch-log-analyzer-and-reporter-on-a-vps)