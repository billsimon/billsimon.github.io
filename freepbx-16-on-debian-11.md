## FreePBX 16 and Debian 11: An Apt Combination

There are many ways to install a FreePBX system; the easiest way is to install it through a FreePBX Distro ISO. But if you want more choices, you have several--FreePBX is an open-source project and works with all supported versions of Asterisk and on various flavors of Linux.

My Linux of choice is Debian, and this past weekend version 11 of the OS was released. In addition, FreePBX version 16 is available in beta, and it uses PHP 7.4, which comes packaged with Debian 11. This seems like a good match. Let's try it out!

In this blog post, we are going to simplify the installation process as much as possible through the use of Debian's "apt" package system. Some benefits of using packages are that they are maintained by the OS maintainers--there's no need to rebuild software from source when there's an update; the command lines are simple; and installations are predictable--software is installed in /usr/bin, /usr/lib, with configuration files in /etc, and so on. Moreover, the software you install from the package system tends to be well-vetted and stable. Some drawbacks to using the package system are that you are limited to the version(s) chosen by the package maintainers and customization may be more difficult. As an example, with Debian 11's apt repository, we are limited to using Asterisk version 16.

Let's walk through the steps to install FreePBX using Debian packages. At the end, you will have a functional FreePBX 16 system with Asterisk 16 at the core.

Start with a base Debian 11 installation (however you get it from your cloud/VPS provider or Debian install disc/ISO) and log in as root or `sudo su -` to become root.

Update to current:

```
apt update && apt upgrade
```

Now load the prerequisite packages including Asterisk and PHP 7.4:

```
apt install -y apache2 mariadb-server mariadb-client php php-curl php-cli php-pdo php-mysql php-pear php-gd php-mbstring php-intl php-bcmath curl sox mpg123 lame ffmpeg sqlite3 git unixodbc sudo dirmngr postfix asterisk odbc-mariadb php-ldap nodejs npm pkg-config libicu-dev
```

FreePBX manages Asterisk, so let's disable it in systemd:

```
systemctl stop asterisk && systemctl disable asterisk
```

Clear out the sample Asterisk configs and leave a bare-bones config for FreePBX to build upon:

```
cd /etc/asterisk
mkdir DIST
mv * DIST
cp DIST/asterisk.conf .
sed -i 's/(!)//' asterisk.conf
touch modules.conf
touch cdr.conf
```

FreePBX requires some adjustments to the default Apache web server configuration:

```
sed -i 's/\(^upload_max_filesize = \).*/\120M/' /etc/php/7.4/apache2/php.ini 
sed -i 's/\(^memory_limit = \).*/\1256M/' /etc/php/7.4/apache2/php.ini
sed -i 's/^\(User\|Group\).*/\1 asterisk/' /etc/apache2/apache2.conf
sed -i 's/AllowOverride None/AllowOverride All/' /etc/apache2/apache2.conf
a2enmod rewrite
systemctl restart apache2
rm /var/www/html/index.html
```

FreePBX uses ODBC to work with the CDR database. Configure it with the following:

```
cat <<EOF > /etc/odbcinst.ini
[MySQL]
Description = ODBC for MySQL (MariaDB)
Driver = /usr/lib/x86_64-linux-gnu/odbc/libmaodbc.so
FileUsage = 1
EOF

cat <<EOF > /etc/odbc.ini
[MySQL-asteriskcdrdb]
Description = MySQL connection to 'asteriskcdrdb' database
Driver = MySQL
Server = localhost
Database = asteriskcdrdb
Port = 3306
Socket = /var/run/mysqld/mysqld.sock
Option = 3
EOF
```

Now we can install FreePBX. I like to stage install files in /usr/local/src:

```
cd /usr/local/src
wget http://mirror.freepbx.org/modules/packages/freepbx/7.4/freepbx-16.0-latest.tgz
tar zxvf freepbx-16.0-latest.tgz 
cd /usr/local/src/freepbx/
./start_asterisk start
./install -n
```

After the FreePBX base installation is done, add the rest of the modules:

```
fwconsole ma installall
```

Note that digiumaddoninstaller, firewall, and xmpp will not install. Firewall will not work without sysadmin (a free but commercial module); xmpp will not work without mongodb. If you want xmpp, install mongodb separately.

Apply what's been done so far:

```
fwconsole chown && fwconsole r
```

Instead of the sound library from the Asterisk package, we're going to use what FreePBX installs:

```
cd /usr/share/asterisk
mv sounds sounds-DIST
ln -s /var/lib/asterisk/sounds sounds
```

Lastly, issue a restart in order to load any Asterisk modules that weren't configured yet:

```
fwconsole restart
```

To have FreePBX start on boot, add a systemd startup script:

```
cat <<EOF > /etc/systemd/system/freepbx.service
[Unit]
Description=FreePBX VoIP Server
After=mariadb.service
[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/sbin/fwconsole start -q
ExecStop=/usr/sbin/fwconsole stop -q
[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable freepbx
```

FreePBX 16 with Asterisk 16 is now installed and you are ready to move on to configuration from the web interface. Browse to http://the-IP-address-of-the-server to continue. What's left?

Don't put off this task: add some network security. The Sangoma firewall module is not available on this installation, so you will probably want to manually configure iptables or ufw to control acccess, unless you have another firewall in front of your server.

UCP's node service doesn't work with the packaged version of nodejs, but [there is a ticket open to solve this](https://issues.freepbx.org/browse/FREEPBX-22742). Keep an eye on module updates to see when this fix comes through.

