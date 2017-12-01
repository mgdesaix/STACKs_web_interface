Recently I got the Stacks web-based interface set up and introduced myself to Apache, PHP, and MySQL in the process...which ended up being a long, time-consuming process with troubleshooting at every step.  In the hopes of shortening the time it takes others to go through this setup, here's the steps I took to make it work on macOS High Sierra.  I compiled information from a variety of sources and guides online. I have tried to make it maneagable even if you haven't worked in Terminal. The general workflow and much of the information was provided by Thierry Gosselin's awesome [tutorial](http://gbs-cloud-tutorial.readthedocs.io/en/latest/09_stacks_web_interface.html) on GBS analysis in the cloud, which was the only detailed guide I could find for setting up the web interface for Stacks. But I ran into plenty of issues so I figured I would provide another resource for people looking to get this set up.    So here's how I did it with some tips and tricks for the problems I ran into...

## Apache

[Apache](https://httpd.apache.org/) is already installed on your mac and you can open it from Terminal.  Many of the commands throughout these steps will require `sudo` in order to bypass the restrictions on the directories being accessed, so make sure you have the admin password.  To start, stop, and restart Apache all you do is the following:

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
And add this line, do not delete anything in **.bash_profile**.
```
export PATH="$PATH:/usr/local/mysql/bin"
```
Save and exit. Enter command:
```
$ source .bash_profile
```
The scripts provided by MySQL can now be read whenever you open a new shell. If you accidentally delete the paths to the core scripts then you won't be able to run even basic commands after sourcing your **.bash_profile**.  If this happens, all you need to do is enter in the Terminal: 

```
export PATH=/usr/bin:/bin:/usr/sbin:/sbin:/usr/local/bin:$PATH
```
And then make sure that is also in your **.bash_profile**.  Ok, now let's change the temporary password that was provided on installation. Enter into the terminal:

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
[Download phpMyAdmin](https://www.phpmyadmin.net/downloads/) and move to your $~/Sites/$ directory as phpmyadmin.
```
$ cd ~/Downloads/
$ wget https://files.phpmyadmin.net/phpMyAdmin/4.7.6/phpMyAdmin-4.7.6-all-languages.zip   # note: 'wget' is from Homebrew
$ unzip phpMyAdmin-4.7.6-all-languages.zip
$ mv ~/Downloads/Admin-4.7.6-all-languages ~/Sites/phpmyadmin
$ mkdir ~/Sites/phpmyadmin/config                             # make configuration folder
$ chmod o+w ~/Sites/phpmyadmin/config                       # to change the permissions
```
### Note: if you don't have $wget$...
Then you need [Homebrew](https://brew.sh/). Check if you have Homebrew
```
$ brew help
```
If you get $command not found$, then you need to install it:
```
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```
Now you can install $wget$
```
$ brew install wget
```
Once you have phpmyadmin downloaded and the 'config' directory set up, go to your browswer and enter:

```
http://localhost/~username/phpmyadmin/setup/    # this works for me, but I've seen the other option listed as well
# or
http://localhost/phpmyadmin/setup/
```
Select 'New Server'

<img src="https://raw.githubusercontent.com/mgdesaix/Stacks_web_interface/master/media/phpmyadmin1.png">

Enter your user information from the MySQL setup and then select 'Apply'

<img src="https://raw.githubusercontent.com/mgdesaix/Stacks_web_interface/master/media/phpmyadmin2.png">

If you have an option to select 'Save', then do that and it will go to the $~/Sites/phpmyadmin/config/$ directory you created...however, I don't have that option and need to 'Download'. Either way, move the file to $~/Sites/pypmyadmin/$

<img src="https://raw.githubusercontent.com/mgdesaix/Stacks_web_interface/master/media/phpmyadmin3.png">

```
$ mv ~/Downloads/config.inc.php ~/Sites/phpmyadmin/config.inc.php
# OR
$ mv ~/Sites/phpmyadmin/config/config.inc.php ~/Sites/phpmyadmin/config.inc.php
$ rmdir ~/Sites/phpmyadmin/config
```
Going to http://localhost/~username/phpmyadmin/ in your web browser should now bring you here:
<img src="https://raw.githubusercontent.com/mgdesaix/Stacks_web_interface/master/media/phpmyadmin4.png">

You have successfully set up phpMyAdmin

## Enable exporting from Stacks to MySQL

Edit the 'stacks_export_notify.pl' script as follows:
```
sudo nano /usr/local/bin/stacks_export_notify.pl
```
```
my $exe_path      = "/usr/local/bin/" . "export_sql.pl";
my $output_path   = "/usr/local/share/stacks/" . "php/export/";
my $url           = "http://stackshost.edu/stacks/export/";
my $local_host    = "localhost";
my $smtp_host     = "localhost";
my $from          = "desaixmg\@mymail.vcu.edu";
```
Save and exit. Set permissions, if your web server user is 'root':

sudo chown root /usr/local/share/stacks/php/export
sudo chmod 775 /usr/local/share/stacks/php/export

## PEAR tools
First install [Pear](https://pear.php.net/index.php), I had to try several options to get this to work:
```
$ curl -O https://pear.php.net/go-pear.phar
$ php -d detect_unicode=0 go-pear.phar
```
Now to configure Pear
1) type 1 and press Enter
2) Enter /usr/local/pear 
3) Press Enter
Then, 
1) Type 4 and press Enter 
2) Enter usr/local/bin
3) Press Enter

Install MDB2 Pear Module
[More Information](http://pear.php.net/package/MDB2/)
```
sudo pear install MDB2-2.5.0b5
```

Install MySQL MDB2 driver - the '--nodeps' option was essential for me
[More information](http://pear.php.net/package/MDB2_Driver_mysql/download)
```
sudo pear install --nodeps MDB2_Driver_mysql-1.5.0b4.tgz
```
The check all of your modules have downloaded, enter
```
$ pear list
```
And you should see them:
```
Installed packages, channel pear.php.net:
=========================================
Package           Version State
Archive_Tar       1.4.3   stable
Console_Getopt    1.4.1   stable
MDB2              2.5.0b5 beta
MDB2_Driver_mysql 1.5.0b4 beta
PEAR              1.10.5  stable
Structures_Graph  1.1.1   stable
XML_Util          1.4.2   stable
```


## Perl modules

Install [cpanminus](https://github.com/miyagawa/cpanminus) to use CPAN modules

```
$ cpan install CPAN # to update CPAN
$ curl -L http://cpanmin.us | perl - --sudo App::cpanminus
```

Install DBI and DBD::mysql
```
$ sudo perl -MCPAN -e 'shell'

get DBI
get DBD::mysql
exit
```
Create alias because MySQL on Mac is installed differently than on Linux
```
$ cd /usr/local
$ sudo mkdir lib  # folder probably already exist
$ cd lib
$ sudo ln -s /usr/local/mysql/lib/*.dylib .    # copy-paste all of this line, including the space and the dot after dylib
```
Install and compile
```
$ cd ~/.cpan/build/DBI*/
$ sudo perl Makefile.PL
$ sudo make
$ sudo make test
$ sudo make install

$ cd ~/.cpan/build/DBD*/
$ sudo perl Makefile.PL --testuser='root' --testpassword='yourpassword' # don't forget to leave the ` and write your password
$ sudo make
$ sudo make test
$ sudo make install
```

Tell php where to find Pear and its module
```
$ sudo cp /private/etc/php.ini.default /private/etc/php.ini
$ sudo nano /private/etc/php.ini
# copy and paste the text below at the end of php.ini file
;***** Added by go-pear
include_path=".:/usr/share/pear"
;*****
crtl-o, enter, crtl-x         # to save and exit nano editor
```
To export data in Excel spreadsheets:
```
$ cd ~/Downloads
$ wget http://search.cpan.org/CPAN/authors/id/J/JM/JMCNAMARA/OLE-Storage_Lite-0.19.tar.gz
$ tar -xvf OLE-Storage_Lite-0.19.tar.gz
$ cd OLE-Storage_Lite-0.19
$ perl Makefile.PL
$ make
$ sudo make install

$ cd ~/Downloads
$ wget http://search.cpan.org/CPAN/authors/id/J/JM/JMCNAMARA/Spreadsheet-WriteExcel-2.40.tar.gz
$ tar -xvf Spreadsheet-WriteExcel-2.40.tar.gz
$ cd Spreadsheet-WriteExcel-2.40
$ perl Makefile.PL
$ make
$ sudo make install

$ cd ..
$ sudo rm -R Spreadsheet-WriteExcel-2.40* OLE-Storage_Lite-0.19*
```

## Upload Stacks output

Create a database and make sure to end name with '_radtags' 
```
$ sudo mysql -u mdesaix -p -e "create database test_radtags" 
```
Prepare the database to receive files
```
$ mysql -u mdesaix -p test_radtags < /usr/local/share/stacks/sql/stacks.sql
```
Load the database.  I'm using a small subset of data to make this go fast, but it can take a long time. You will want to [explore](http://catchenlab.life.illinois.edu/stacks/comp/load_radtags.php) the 'load_radtags' settings when you are running this for your data
```
$ D="-D test_radtags"
$ p="-p /Users/mdesaix/Desktop/MySQL" # path to your Stacks output files

$ load_radtags.pl $D $p -b 1 -c -B -e "this is a test"

index_radtags.pl -D test_radtags -c . # index samples
```
Now when you go to http://localhost/stacks

You should see

<img src="https://raw.githubusercontent.com/mgdesaix/Stacks_web_interface/master/media/localhost_stacks.png">


## Finished!

If you need help see:

Thierry Gosselin's awesome tutorial: http://gbs-cloud-tutorial.readthedocs.io/en/latest/09_stacks_web_interface.html
Set up a local host on High Sierra: https://websitebeaver.com/set-up-localhost-on-macos-high-sierra-apache-mysql-and-php-7-with-sslhttps

Stacks Google User Group: https://groups.google.com/forum/#!forum/stacks-users

Good luck!
