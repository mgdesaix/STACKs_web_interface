# Setting up the Stacks web interface

Recently I got the STACKs web-based interface set up and introduced myself to Apache, PHP, and MySQL in the process...which ended up being a long, time-consuming process.  In the hopes of shortening the time it takes others to go through this setup, here's the steps I took to make it work on macOS High Sierra.  If you haven't worked in Terminal before be careful, you can ruin everything pretty easily.

## Apache

Apache is already installed on your mac and you can open it from Terminal.  Many of the commands throughout these steps will require `sudo` in order to bypass the restrictions on the directories being accessed, so make sure you have the admin password.  To start, stop, and restart Apache all you do is the following:

```
$ sudo apachectl start
$ sudo apachectl stop
$ sudo apachectl restart
```
To check the version, you refer to Apache by its process name `httpd` (short for "HTTP daemon"):
```
$ sudo httpd -v
Server version: Apache/2.4.27 (Unix)
Server built:   Jul 15 2017 15:41:46
```
After starting Apache, go to your browser and type in:
```
http://localhost
```
And you should see "It works!":

![It works](https://i.stack.imgur.com/K3Virm.png)

If you don't, then stop and resolve the issue.  To check the configuration use:
```
$ apachectl configtest
```
One problem I ran into was having two versions (one from Homebrew) of Apache on the sytem I was using and I went back and removed the version from Homebrew.  To check any issues with Apache and whether it is properly loading, try checking out the error and access logs:
```
$ cd private/var/log/apache/
$ ls
access_log	error_log
```

Now you need to create the root directory for $localhost$.  
```
$ cd ~
$ mkdir Sites
```
Figure out your account:
```bash
$ whoami   # gives the name of your account
```
Now you need to edit the apache user configuration file, we'll use nano

```
$ cd /etc/apache2/users
$ ls                        # check out the list of users 
$ sudo nano username.conf   # use your account for username and open up the .conf file
```

Add the content as follows and replace *username* with your account
```
<Directory "/Users/username/Sites/">
AllowOverride All
Options Indexes MultiViews FollowSymLinks
Require all granted
</Directory>

<Directory "/usr/local/share/stacks/php">
Require all granted
</Directory>

Alias /stacks "/usr/local/share/stacks/php"
```

Save, exit and then put restrictions on the file:

```
$ crtl-o                                           # save
$ enter
$ crtl-x                                           # exit
$ sudo chmod 644 /etc/apache2/users/username.conf  # put restriction on the file, change username
$ ls -l                                            # view the permissions
```
To make the PHP files in STACKs visible to the Apache server you need to create a **stacks.conf** file

```
$ cd /etc/apache2/                  
$ sudo mkdir conf.d                  # make new directory
$ sudo nano ./conf.d/stacks.conf     # create stacks.conf file in directory
```

In this file, enter:

```
<Directory "/usr/local/share/stacks/php">
               Require all granted
</Directory>

Alias /stacks "/usr/local/share/stacks/php"
```
```
$ crtl-o                                           
$ enter
$ crtl-x                                           
$ sudo chmod 644 /etc/apache2/conf.d/stacks.conf
```

Now to edit the main configuration file for Apache, this will authorize use of PHP and necessary modules. In case something goes wrong with this edit you can find an original version to copy back over in $/etc/apache2/original$.  To edit:

```
$ cd /etc/apache2/
$ sudo nano ./httpd.conf
```

Uncomment all of the following lines by removing the '#' (In order to make sure you find everything, you can search in nano with `control`+`w`):
```
LoadModule authz_core_module libexec/apache2/mod_authz_core.so
LoadModule authz_host_module libexec/apache2/mod_authz_host.so
LoadModule userdir_module libexec/apache2/mod_userdir.so
LoadModule include_module libexec/apache2/mod_include.so
LoadModule rewrite_module libexec/apache2/mod_rewrite.so
LoadModule php7_module libexec/apache2/libphp7.so
Include /private/etc/apache2/extra/httpd-userdir.conf
```

Also in this file you need to change a couple of other things, find **.htaccess** and change **AllowOverride** from 'None' to 'All' to: 
```
# AllowOverride controls what directives may be placed in .htaccess files.
    # It can be "All", "None", or any combination of the keywords:
    #   AllowOverride FileInfo AuthConfig Limit
    #
    AllowOverride All
```

Also, find the following section and change to 'Require all granted'
```
# Deny access to the entirety of your server's filesystem. You must
# explicitly permit access to web content directories in other
# <Directory> blocks below.
#
<Directory />
    Require all granted
</Directory>
```

Save and exit (Ctl-O, enter, Ctl-x)

Lastly, one last configuration file to modify:
```
$ sudo nano /etc/apache2/extra/httpd-userdir.conf
```
Uncomment:
```
Include /private/etc/apache2/users/*.conf
```
Save and exit (Ctl-O, enter, Ctl-x)

Restart Apache and it is good to go

```
$ sudo apachectl restart
```

## PHP

PHP is already on your Mac and you enabled it in the step above by uncommenting $LoadModule php7_module libexec/apache2/libphp7.so$.

Let's test PHP. Go to $~/Sites/$ and create a PHP file
```
$ cd ~/Sites/
$ sudo nano ./phpinfo.php
```
Within this file add:
```
<?php
phpinfo();
?>
```
Save and exit.  Now check it out in your browser:

```
http://localhost/~username/phpinfo.php    # change username
```
![](https://raw.githubusercontent.com/mgdesaix/Stacks_web_interface/master/media/PHP.png)

## MySQL

MySQL is the database system used for the web interface and it is not provided with High Sierra, but you can [download it here](https://dev.mysql.com/downloads/mysql/).  When it asks you to 'Login' or 'Sign Up' you can click the 'No thanks, just start my download' at the bottom.  Go through the installation and make sure to COPY down the password they give you!!!

<img src="https://raw.githubusercontent.com/mgdesaix/Stacks_web_interface/master/media/MySQL_password.png" width="550" height="400">

Once you have MySQL installed, start it up in terminal
```
$ sudo /usr/local/mysql/support-files/mysql.server start
```

In order to use commands from newly installed programs you need to edit your shell startup script, the **.bash_profile**, to point toward the scripts provided by the new program instead of having to provide the full path everytime you wish to use them.  In this instance, MySQL downloads scripts here: `/usr/local/mysql/bin` and you will enable the **.bash_profile** to find them.
```
$ cd
$ sudo nano .bash_profile
```
And add this line
```
export PATH="$PATH:/usr/local/mysql/bin"
```
Save and exit. Enter command:
```
$ source .bash_profile
```
The scripts provided by MySQL can now be read whenever you open a new shell. Ok, now let's change the temporary password that was provided on installation. Enter into the terminal:

```
$ mysql -u root -p
#(enter the temporary password)
```
MySQL is open and says "Welcome to the MySQL monitor...".
Now enter the following and set a new password:
```
SET PASSWORD FOR 'root'@'localhost' = PASSWORD('the_password_you_set');
```
Quit with:
```
\q
```
Edit the MySQL configuration file with your user information:
```
$ cd /usr/local/share/stacks/sql/
$ sudo cp mysql.cnf.dist mysql.cnf
$ sudo nano /usr/local/share/stacks/sql/mysql.cnf
```
And change to your database username and password:
```
user=mdesaix               #your_username
password=my_password       # your_password 
host=localhost
port=3306
```
Then log back into mysql and grant access to the user information you set in mysql.cnf:
```
$ mysql -u root -p
```
```
GRANT ALL ON *.* TO 'mdesaix'@'localhost' IDENTIFIED BY 'my_password';
\q
```
Then we need to allow PHP to access the MySQL database by editing the PHP configuration file, **constants.php.dist**. You will provide the database username and password, and you will also the $$mysql_bin$ from "/usr/bin/mysql" to "/usr/local/mysql/bin/mysql".
```
sudo cp /usr/local/share/stacks/php/constants.php.dist /usr/local/share/stacks/php/constants.php
sudo nano /usr/local/share/stacks/php/constants.php
```

```
// Credentials to access Stacks MySQL databases
//
$db_user     = "mdesaix";
$db_pass     = "my_password";
$db_host     = "localhost";

//
// Path to the MySQL client program
//
$mysql_bin = "/usr/local/mysql/bin/mysql";
```
Alright, lastly you need to fix a socket error because MySQL places the socket in "/tmp" and OSX is looking for it in "/var/mysql".

```
sudo mkdir /var/mysql
sudo ln -s /tmp/mysql.sock /var/mysql/mysql.sock
```
### Intermission
Breathe. If you've had no problems this far, that's awesome.  If you've had some, that's normal.

## phpMyAdmin


## PEAR tools

## Perl modules

## Upload Stacks output

## Finished!

