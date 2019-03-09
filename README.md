# How to install Horde Webmail on Ubuntu 18.04 LTS (the quick and dirty way)


## Infrastructural prerequisites

* a server (physical or virtual) with root access
* an imap mail account (on this server or at some other provider)


## Preparations

Upgrade the OS and enable time sync
```
apt update && apt full-upgrade -y

apt install -y ntp
systemctl enable ntp && systemctl restart ntp
```

Install necessary packages
```
apt install -y apache2 vim less tree php-memcache apache2 php-pecl-http php-pear php7.2 php7.2-mysql mysql-server php7.2-mbstring php7.2-gd php7.2-tidy php-mime-type php7.2-intl php7.2-curl php-geoip php-mbstring php-imagick php-horde-lz4 php7.2-dev atop
```

Secure mysql installation
```
mysql_secure_installation
```

Create Horde database
```
mysql -u root -p
CREATE DATABASE horde_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
grant all on horde_db.* to 'horde_user'@'localhost' identified by 'somepass';
flush privileges;
exit
```

## Install Horde base
```
pear channel-update pear.php.net
pear channel-discover pear.horde.org
pear install horde/horde_role

# set install directory (it will be referred as INSTALLDIR)
pear run-scripts horde/horde_role

pear install -a -B horde/horde
chown www-data:www-data INSTALLDIR/horde -R
cd INSTALLDIR/horde/config
cp conf.php.dist conf.php

# enable test page
sed -i '61s/true/false/' conf.php

# Horde 5.2.22 is not certified for php7 yet, small adjustment has to be done
sed -i 's/\&new/new/g' /usr/share/php/File/Fstab.php
```

Open test page and fix the possbile issues (not all of them are blocker or mandatory): http://SERVERNAME.FQDN/horde/test.php


### Configure Horde

#### Database connection

Open http://SERVERNAME.FQDN/horde/admin/config/config.php?app=horde

Click "Database"

Set driver to "mysqli" and enter DB connection parameters.

Click "Generate Horde Configuration" button at the bottom.

Import DB schema
```
/usr/bin/horde-db-migrate
```

#### Set up authentication

Open http://SERVERNAME.FQDN/horde/admin/config/config.php?app=horde

Click "Authentication"

Add all admin users (with mail addresses) to '$conf[auth][admins]'

Set '$conf[auth][driver]' to 'IMAP' (port: 143, tls ON)

Click "Generate Horde Configuration" button at the bottom.

Log out.

Log in with one of the admin users configured above.


## Install IMP (mail client)

### Install IMP
```
pear install -a -B horde/imp
/usr/bin/horde-db-migrate imp

chown www-data:www-data INSTALLDIR/horde/imp -R
```

Open http://SERVERNAME.FQDN/horde/admin/config/config.php?app=imp

Set 4K key size under "PGP settings"

Generate the config file.

#### Create the custom backend config with EXTERNAL-MAIL-SERVER.FQDN
```
cd INSTALLDIR/horde/imp/config
rm -rfv backends.local.php
echo "<?php" >> backends.local.php
echo "" >> backends.local.php
echo "\$servers['imap'] = array(" >> backends.local.php
echo "'disabled' => false," >> backends.local.php
echo "'name' => 'IMAP Server'," >> backends.local.php
echo "'hostspec' => 'EXTERNAL-MAIL-SERVER.FQDN'," >> backends.local.php
echo "'hordeauth' => true," >> backends.local.php
echo "'protocol' => 'imap'," >> backends.local.php
echo "'port' => 143," >> backends.local.php
echo "'secure' => 'tls'," >> backends.local.php
echo ");" >> backends.local.php

chown www-data:www-data INSTALLDIR/horde/imp -R
```

## Install Kronolith (calendar)

### Install Kronolith
```
pear install -a horde/kronolith
/usr/bin/horde-db-migrate kronolith
cd INSTALLDIR/horde
chown www-data:www-data kronolith -R
```

Open http://SERVERNAME.FQDN/horde/admin/config/config.php?app=horde

Click 'Shares'

Driver: SQL

Generate the config file


Open http://SERVERNAME.FQDN/horde/admin/config/config.php?app=kronolith

Click 'Maps'

Select a map provider (optional).

Click 'Update all DB schemas' button on the main screen.

## Test the new setup

### Disable test page
```
cd INSTALLDIR/horde/config
sed -i '61s/true/false/' conf.php
```

Login to the new installation at http://SERVERNAME.FQDN/horde/login.php

