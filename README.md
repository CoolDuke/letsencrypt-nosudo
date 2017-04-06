#Let's Encrypt Without Sudo but still unattended

The [Let's Encrypt Without Sudo](https://github.com/diafygi/letsencrypt-nosudo)
is a fantastic program for [Let's Encrypt](https://letsencrypt.org/) to be sure
to not let the whole thing run as root.

The problem is, that you have to do all steps manually every three months. So this
the modified version that fully runs unattended. But note that the scripts will
have access to your private keys.

The sign_csr.py script is now configurable by a file named letsencrypt-nosudo.conf.
It contains information about the certifacte locations

There is only one part of the project that needs root permissions, the part that
copies the created certificates to the webserver configuration directory and restarts
the webserver afterwards. It's just a small bash script comparing the MD5 sums of
the existing and the newly generated certificates.

# Setup
We will install the toolset in /opt/letsencrypt-nosudo.

## Get the scripts
Login as root or create a root shell with sudo -s or su -.

Create a system user for the certificate generation:
```bash
groupadd letsencrypt
useradd -s /bin/bash -d /home/letsencrypt -m -g letsencrypt letsencrypt
```

Get the code, create needed directories and set the user rights (assuming your
webserver is running with group www-data):
```bash
cd /opt
git clone https://github.com/coolduke/letsencrypt-nosudo-unattended /opt/letsencrypt-nosudo
mkdir -pm750 /opt/letsencrypt-nosudo/{certs/account,tmp,www}
chown -R letsencrypt.letsencrypt /opt/letsencrypt-nosudo/
chgrp www-data /opt/letsencrypt-nosudo/{,www}
```

## Setup Apache 2.4
This will add an alias called
/.well-known/acme-challenge to all virtual hosts if there is no redirect defined.
These commands should work in the default configuration of the latest Debian and Ubuntu
versions. You may need to modify it to let it run properly on your setup:
```bash
ln -s /opt/letsencrypt-nosudo/webserver-config/letsencrypt-apache2.4.conf /etc/apache2/conf-available/letsencrypt.conf
a2enconf letsencrypt
systemctl reload apache2
```

## Setup Nginx
Since nginx does not support global aliases, you will have to enable it for every virtual host.
Using the Debian way, a snippet directory is being used to include small parts of configuration
for multiple virtual hosts. So we create a link to the snippet at that directory first:
```bash
ln -s /opt/letsencrypt-nosudo/webserver-config/letsencrypt-nginx-snippet.conf /etc/nginx/snippets/letsencrypt.conf
```
Now add the following line to every HTTP server block you want to have a certificate for:
```bash
include "/etc/nginx/snippets/letsencrypt.conf";
```
And reload nginx:
```bash
systemctl reload nginx
```

## Test challenge directory
Check the availability of the challenge directory by putting a test file into the target
directory and test it in your browser with all URLs you want to get certificates for:
```bash
echo "Hello World" > /opt/letsencrypt-nosudo/www/test.txt
#Open this URL in your browser: http://www.yourdomain.de/.well-known/acme-challenge/test.txt
```
Note that this test has to be successful, otherwise your domains cannot be authenticated.

## Configure letsencrypt-nosudo
Login as user letsencrypt:
```bash
su - letsencrypt
```

Configure the script - if you use the paths from this setup guide you just have to
set your mail address and the domains you want to use here. If you already have a user
key-pair for Letsencrypt, copy it to the configured location:
```bash
cd /opt/letsencrypt-nosudo
cp letsencrypt-nosudo.conf.skel letsencrypt-nosudo.conf
editor letsencrypt-nosudo.conf
```

If you haven't use Letsencrypt so far, you need to create a user key-pair to authenticate
with their servers and set access rights for them:
```bash
openssl genrsa 4096 > /opt/letsencrypt-nosudo/certs/user.key
openssl rsa -in /opt/letsencrypt-nosudo/certs/user.key -pubout > /opt/letsencrypt-nosudo/certs/user.pub
chmod 660 /opt/letsencrypt-nosudo/certs/user.{key,pub}
```

Test the script - if this is your first run it will create all the certificates you requested.
Be sure you still are logged in as user letsencrypt:
```bash
python check_and_update_crt.py
```

If everthing is fine, you may add a cronjob - be sure you still are logged in as user letsencrypt:
```bash
crontab -e
```
Add an entry like this to let the check run every day at 12:24 am. This checks if a certificate has
to be renewed and does it for you:
```bash
24 0 * * * /usr/bin/python /opt/letsencrypt-nosudo/check_and_update_crt.py > /dev/null
```
To get more information about the script and the original author, please refer to https://github.com/diafygi/letsencrypt-nosudo.

# Automatic Apache configuration script
The bash script update_apache_certs.sh checks if a certificate has been renewed,
copies it to the Apache configuration directory and restarts the webserver to load it.

It is supposed to run as root cronjob, but you can use the attached sudoers.d file to add the script
to the crontab of our created letsencrypt user.

Login as root or create a root shell with sudo -s or su -.

You might want to prevent editing of the script by any normal user:
```bash
chown root.root /opt/letsencrypt-nosudo/update_apache_certs.sh
chmod 600 /opt/letsencrypt-nosudo/update_apache_certs.sh
```

Edit root's crontab:
```bash
crontab -e
```
Add an entry like this to let the check run every day at 01:24 am, leave a little time gap between the
letsencrypt cronjob we defined before:
```bash
24 1 * * * /bin/bash /opt/letsencrypt-nosudo/update_apache_certs.sh
```

# Automatic Nginx configuration script
The bash script update_nginx_certs.sh checks if a certificate has been renewed,
copies it to the Nginx configuration directory and restarts the webserver to load it.

It is supposed to run as root cronjob, but you can use the attached sudoers.d file to add the script
to the crontab of our created letsencrypt user.

Login as root or create a root shell with sudo -s or su -.

You might want to prevent editing of the script by any normal user:
```bash
chown root.root /opt/letsencrypt-nosudo/update_nginx_certs.sh
chmod 600 /opt/letsencrypt-nosudo/update_nginx_certs.sh
```

Edit root's crontab:
```bash
crontab -e
```
Add an entry like this to let the check run every day at 01:24 am, leave a little time gap between the
letsencrypt cronjob we defined before:
```bash
24 1 * * * /bin/bash /opt/letsencrypt-nosudo/update_nginx_certs.sh
```

# TODO
The revoke script had not been touched yet, It should also use the configuration file in the future.
