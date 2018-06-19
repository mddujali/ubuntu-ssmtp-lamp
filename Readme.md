# Setup SSMTP in LAMP Stack (Ubuntu 16.04 or Higher)

## Requirements
- LAMP Stack
- SSMTP

## Setup LAMP stack

### Step 1: Install Apache and Allow in Firewall

The Apache web server is among the most popular web servers in the world. It's well-documented, and has been in wide use for much of the history of the web, which makes it a great default choice for hosting a website.

We can install Apache easily using Ubuntu's package manager, apt. A package manager allows us to install most software pain-free from a repository maintained by Ubuntu. You can learn more about how to use apt here.

For our purposes, we can get started by typing these commands:
```
$ sudo apt-get update
$ sudo apt-get install apache2
```

Since we are using a sudo command, these operations get executed with root privileges. It will ask you for your regular user's password to verify your intentions.

Once you've entered your password, apt will tell you which packages it plans to install and how much extra disk space they'll take up. Press Y and hit Enter to continue, and the installation will proceed.

#### Set Global ServerName to Suppress Syntax Warnings

Next, we will add a single line to the /etc/apache2/apache2.conf file to suppress a warning message. While harmless, if you do not set ServerName globally, you will receive the following warning when checking your Apache configuration for syntax errors:
```console
$ sudo apache2ctl configtest
```

Output:
```console
AH00558: apache2: Could not reliably determine the server's fully qualified domain name, using 127.0.1.1. Set the 'ServerName' directive globally to suppress this message
Syntax OK
```

Open up the main configuration file with your text edit:
```console
$ sudo nano /etc/apache2/apache2.conf
```

Inside, at the bottom of the file, add a ServerName directive, pointing to your primary domain name. If you do not have a domain name associated with your server, you can use your server's public IP address:
```console
ServerName server_domain_or_IP // localhost
```
Save and close the file when you are finished.

Next, check for syntax errors by typing:
```console
$ sudo apache2ctl configtest
```
Since we added the global ServerName directive, all you should see is:
```console
Syntax OK
```

Make host remotely accessible.
```console
<VirtualHost *:80>
    ...
    <Directory "/path/to/your/directory">
        	Options Indexes FollowSymLinks
        	AllowOverride all
        	Require all granted
        	Order Deny,Allow
        	Allow from all
    </Directory>
</VirtualHost>
```

Restart Apache to implement your changes:
```
$ sudo systemctl restart apache2
```

#### Adjust the Firewall to Allow Web Traffic

Next, assuming that you have followed the initial server setup instructions to enable the UFW firewall, make sure that your firewall allows HTTP and HTTPS traffic. You can make sure that UFW has an application profile for Apache like so:
```console
$ sudo ufw app list
```

Output:
```console
Available applications:
Apache
Apache Full
Apache Secure
OpenSSH
```

If you look at the Apache Full profile, it should show that it enables traffic to ports 80 and 443:
```
$ sudo ufw app info "Apache Full"
```

Output:
```console
Profile: Apache Full
Title: Web Server (HTTP,HTTPS)
Description: Apache v2 is the next generation of the omnipresent Apache web
server.

Ports:
    80,443/tcp
```

Allow incoming traffic for this profile:
```console
sudo ufw allow in "Apache Full"
```

You can do a spot check right away to verify that everything went as planned by visiting your server's public IP address in your web browser (see the note under the next heading to find out what your public IP address is if you do not have this information already):
```console
http://your_server_IP_address // http://localhost
```

#### How To Find your Server's Public IP Address

If you do not know what your server's public IP address is, there are a number of ways you can find it. Usually, this is the address you use to connect to your server through SSH.

From the command line, you can find this a few ways. First, you can use the iproute2 tools to get your address by typing this:
```console
    $ ip addr show eth0 | grep inet | awk '{ print $2; }' | sed 's/\/.*$//'
```

This will give you two or three lines back. They are all correct addresses, but your computer may only be able to use one of them, so feel free to try each one.

An alternative method is to use the curl utility to contact an outside party to tell you how it sees your server. You can do this by asking a specific server what your IP address is:

```console
$ sudo apt-get install curl
$ curl http://icanhazip.com
```

Regardless of the method you use to get your IP address, you can type it into your web browser's address bar to get to your server.

### Install MySQL

Now that we have our web server up and running, it is time to install MySQL. MySQL is a database management system. Basically, it will organize and provide access to databases where our site can store information.

Again, we can use apt to acquire and install our software. This time, we'll also install some other "helper" packages that will assist us in getting our components to communicate with each other:
```console
$ sudo apt-get install mysql-server
```

> **Note:** In this case, you do not have to run sudo apt-get update prior to the command. This is because we recently ran it in the commands above to install Apache. The package index on our computer should already be up-to-date.

Again, you will be shown a list of the packages that will be installed, along with the amount of disk space they'll take up. Enter Y to continue.

During the installation, your server will ask you to select and confirm a password for the MySQL "root" user. This is an administrative account in MySQL that has increased privileges. Think of it as being similar to the root account for the server itself (the one you are configuring now is a MySQL-specific account, however). Make sure this is a strong, unique password, and do not leave it blank.

When the installation is complete, we want to run a simple security script that will remove some dangerous defaults and lock down access to our database system a little bit. Start the interactive script by running:
```console
$ mysql_secure_installation
```

You will be asked to enter the password you set for the MySQL root account. Next, you will be asked if you want to configure the **VALIDATE PASSWORD PLUGIN**.

> **Warning:** VALIDATE PASSWORD PLUGIN can be used to test passwords
> and improve security. It checks the strength of password
> and allows the users to set only those passwords which are
> secure enough. Would you like to setup VALIDATE PASSWORD plugin?

> Press y|Y for Yes, any other key for No:

You'll be asked to select a level of password validation. Keep in mind that if you enter 2, for the strongest level, you will receive errors when attempting to set any password which does not contain numbers, upper and lowercase letters, and special characters, or which is based on common dictionary words.
```console
There are three levels of password validation policy:

LOW    Length >= 8
MEDIUM Length >= 8, numeric, mixed case, and special characters
STRONG Length >= 8, numeric, mixed case, special characters and dictionary                  file

Please enter 0 = LOW, 1 = MEDIUM and 2 = STRONG: 1
```

For the rest of the questions, you should press Y and hit the Enter key at each prompt. This will remove some anonymous users and the test database, disable remote root logins, and load these new rules so that MySQL immediately respects the changes we have made.

At this point, your database system is now set up and we can move on.

### Step 3: Install PHP

PHP is the component of our setup that will process code to display dynamic content. It can run scripts, connect to our MySQL databases to get information, and hand the processed content over to our web server to display.

We can once again leverage the apt system to install our components. We're going to include some helper packages as well, so that PHP code can run under the Apache server and talk to our MySQL database:
```console
$ sudo apt-get install php libapache2-mod-php php-mcrypt php-mysql
```

This should install PHP without any problems. We'll test this in a moment.

In most cases, we'll want to modify the way that Apache serves files when a directory is requested. Currently, if a user requests a directory from the server, Apache will first look for a file called index.html. We want to tell our web server to prefer PHP files, so we'll make Apache look for an index.php file first.

To do this, type this command to open the dir.conf file in a text editor with root privileges:
```console
$ sudo nano /etc/apache2/mods-enabled/dir.conf
```

```console
<IfModule mod_dir.c>
    DirectoryIndex index.html index.cgi index.pl index.php index.xhtml index.htm
</IfModule>
```

We want to move the PHP index file highlighted above to the first position after the DirectoryIndex specification, like this:
```console
<IfModule mod_dir.c>
    DirectoryIndex index.php index.html index.cgi index.pl index.xhtml index.htm
</IfModule>
```

When you are finished, save and close the file by pressing Ctrl-X. You'll have to confirm the save by typing Y and then hit Enter to confirm the file save location.

After this, we need to restart the Apache web server in order for our changes to be recognized. You can do this by typing this:
```console
$ sudo systemctl restart apache2 
```

We can also check on the status of the apache2 service using systemctl:
```console
$ sudo systemctl status apache2
```

Output:
```console
apache2.service - LSB: Apache2 web server
   Loaded: loaded (/etc/init.d/apache2; bad; vendor preset: enabled)
   Drop-In: /lib/systemd/system/apache2.service.d
           └─apache2-systemd.conf
   Active: active (running) since Wed 2016-04-13 14:28:43 EDT; 45s ago
   Docs: man:systemd-sysv-generator(8)
   Process: 13581 ExecStop=/etc/init.d/apache2 stop (code=exited, status=0/SUCCESS)
   Process: 13605 ExecStart=/etc/init.d/apache2 start (code=exited, status=0/SUCCESS)
   Tasks: 6 (limit: 512)
   CGroup: /system.slice/apache2.service
           ├─13623 /usr/sbin/apache2 -k start
           ├─13626 /usr/sbin/apache2 -k start
           ├─13627 /usr/sbin/apache2 -k start
           ├─13628 /usr/sbin/apache2 -k start
           ├─13629 /usr/sbin/apache2 -k start
           └─13630 /usr/sbin/apache2 -k start

Apr 13 14:28:42 ubuntu-16-lamp systemd[1]: Stopped LSB: Apache2 web server.
Apr 13 14:28:42 ubuntu-16-lamp systemd[1]: Starting LSB: Apache2 web server...
Apr 13 14:28:42 ubuntu-16-lamp apache2[13605]:  * Starting Apache httpd web server apache2
Apr 13 14:28:42 ubuntu-16-lamp apache2[13605]: AH00558: apache2: Could not reliably determine the server's fully qualified domain name, using 127.0.1.1. Set the 'ServerNam
Apr 13 14:28:43 ubuntu-16-lamp apache2[13605]:  *
Apr 13 14:28:43 ubuntu-16-lamp systemd[1]: Started LSB: Apache2 web server.
```

#### Install PHP Modules

To enhance the functionality of PHP, we can optionally install some additional modules.

To see the available options for PHP modules and libraries, you can pipe the results of apt-cache search into less, a pager which lets you scroll through the output of other commands:
```console
$ apt-cache search php- | less
```

Use the arrow keys to scroll up and down, and q to quit.

The results are all optional components that you can install. It will give you a short description for each:
```console
libnet-libidn-perl - Perl bindings for GNU Libidn
php-all-dev - package depending on all supported PHP development packages
php-cgi - server-side, HTML-embedded scripting language (CGI binary) (default)
php-cli - command-line interpreter for the PHP scripting language (default)
php-common - Common files for PHP packages
php-curl - CURL module for PHP [default]
php-dev - Files for PHP module development (default)
php-gd - GD module for PHP [default]
php-gmp - GMP module for PHP [default]
…
:
```

To get more information about what each module does, you can either search the internet, or you can look at the long description of the package by typing:

```console
$ apt-cache show package_name
```

There will be a lot of output, with one field called Description-en which will have a longer explanation of the functionality that the module provides.

For example, to find out what the php-cli module does, we could type this:
```console
$ apt-cache show php-cli
```

Along with a large amount of other information, you'll find something that looks like this:
```console
Description-en: command-line interpreter for the PHP scripting language (default)
This package provides the /usr/bin/php command interpreter, useful for
testing PHP scripts from a shell or performing general shell scripting tasks.
.
PHP (recursive acronym for PHP: Hypertext Preprocessor) is a widely-used
open source general-purpose scripting language that is especially suited
for web development and can be embedded into HTML.
.
This package is a dependency package, which depends on Debian's default
PHP version (currently 7.0).
```

If, after researching, you decide you would like to install a package, you can do so by using the apt-get install command like we have been doing for our other software.

If we decided that php-cli is something that we need, we could type:
```console
$ sudo apt-get install php-cli
```

If you want to install more than one module, you can do that by listing each one, separated by a space, following the apt-get install command, like this:
```console
$ sudo apt-get install package1 package2 ...
```

At this point, your LAMP stack is installed and configured. We should still test out our PHP though.

### Step 4: Test PHP Processing on your Web Server

In order to test that our system is configured properly for PHP, we can create a very basic PHP script.

We will call this script info.php. In order for Apache to find the file and serve it correctly, it must be saved to a very specific directory, which is called the "web root".

In Ubuntu 16.04, this directory is located at /var/www/html/. We can create the file at that location by typing:

```console
$ sudo nano /var/www/html/info.php 
```

This will open a blank file. We want to put the following text, which is valid PHP code, inside the file:
```php
<?php
phpinfo();
```

When you are finished, save and close the file.

Now we can test whether our web server can correctly display content generated by a PHP script. To try this out, we just have to visit this page in our web browser. You'll need your server's public IP address again.

The address you want to visit will be:
```plain
http://your_server_IP_address/info.php // http://localhost/info.php
```

## Setup SSMTP

### Step 1: Installation

ssmtp is a send-only sendmail emulator for machines which normally pick their mail up from a centralized mailhub (via pop, imap, nfs mounts or other means). It provides the functionality required for humans and programs to send mail via the standard or /usr/bin/mail user agents.
```console
$ sudo apt-get ssmtp
```

### Step 2: Configuration

SSMTP has two configuration files;
- /etc/ssmtp/ssmtp.conf - configuration file
- /etc/ssmtp/revaliases - reverse aliases file

/etc/ssmtp/ssmtp.conf
```console
#
# Config file for sSMTP sendmail
#
# The person who gets all mail for userids &lt; 1000
# Make this empty to disable rewriting.
root=me@gmail.com
```

This is used to send all emails sent by system users as from the above email address so they can be replied to. e.g. If you are using mod-php then all mail will be sent out as www-data@web.server, you can change this to webmaster@domain by setting the above configuration value.
```console
# The place where the mail goes. The actual machine name is required no
# MX records are consulted. Commonly mailhosts are named mail.domain.com
# Use mail.domain.com:PORT if you want to specify PORT (e.g. mail.server:587)
mailhub=smtp.gmail.com:587
```

This is your actual mail server which acts as the MTA.
```console
# The full hostname
hostname=localhost
```

Very important, if you set this to NO (which seems to be the default) all your messages will be sent out as www-data@web.server and cannot be overridden by "From: " fields.

> **Note:** www-data is the user executing the PHP mail() function which is usually the apache daemon user unless you are using suExec.

If your SMTP server requires authentication and/or TLS, you'll have to set the following configuration settings.
```console
# Are users allowed to set their own From: address?
# YES - Allow the user to specify their own From: address
# NO - Use the system generated From: address
FromLineOverride=YES

# The address where the mail appears to come from for user authentification.
rewriteDomain=gmail.com

# Email credentials
AuthUser=me@gmail.com
AuthPass=my_strong_password
AuthMethod=LOGIN

# Use SSL/TLS before starting negotiation
UseSTARTTLS=YES
#UseTLS=YES
```

/etc/ssmtp/revaliases
```console
# Format:       local_account:outgoing_address:mailhub
# Example: root:your_login@your.domain:mailhub.your.domain[:port]

root:me@gmail.com:smtp.gmail.com:25
```

This file is used to configure Reverse Alases and only works when a email "From: " field is not specified or FromLineOverride=NO

### Step 3: Testing

```console
------------- mail.txt ----------------
To: someone@example.com
From: "Do NOT Reply"
Subject: Testing SSMTP

This is a test message sent from SSMTP.
```

If you remove the "From:" field from your message SSMTP will map the *nix username to an email address (username@host). You can change the mapping using the /etc/ssmtp/revaliases file as above.
```
$ /usr/sbin/ssmtp -v me@gmail.com < mail.txt
```

> **Note:** At this point ssmtp is configured properly and PHP mail() function should also be working. To make it more precise we can change the PHP sendmail_path to /usr/sbin/ssmtp -t

```console
$ sudo nano /etc/php/7.0/apache2/php.ini
```

Now make sure that your php.ini has correct sendmail_path. It should read as:
```console
    sendmail_path = /usr/sbin/ssmtp -t
```

#### Test PHP

```php
------------- mail.php ----------------

<?php
$to = '"Somename Lastname" <<a href="mailto:someone@email.com">someone@email.com</a>>';
$subject = 'PHP mail tester';
$message = 'This message was sent via PHP!' . PHP_EOL .
           'Some other message text.' . PHP_EOL . PHP_EOL .
           '-- signature' . PHP_EOL;
$headers = 'From: "From Name" <<a href="mailto:from@email.dom">from@email.dom</a>>' . PHP_EOL .
           'Reply-To: <a href="mailto:reply@email.com">reply@email.com</a>' . PHP_EOL .
           'Cc: "CC Name" <<a href="mailto:cc@email.dom">cc@email.dom</a>>' . PHP_EOL .
           'X-Mailer: PHP/' . phpversion();

if (mail($to, $subject, $message, $headers))
  echo 'mail() Success!';
else
  echo 'mail() Failed!';
```

## How to Fix - send-mail: Authorization failed

1. Log into your google email account and then go to this link: [https://www.google.com/settings/security/lesssecureapps](https://www.google.com/settings/security/lesssecureapps) and set "Access for less secure apps" to ON. Test to see if your issue is resolved. If it isn't resolved continue to Step #2.

2. Go to [https://support.google.com/accounts/answer/6009563](https://support.google.com/accounts/answer/6009563) (Titled: "Password incorrect error"). This page says "There are several reasons why you might see a “Password incorrect” error (aka 534-5.7.14) when signing in to Google using third-party apps. In some cases even if you type your password correctly." This page gives 4 suggestions of things to try.
