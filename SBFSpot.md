# Overview

This document will outline the steps on how to install [SBFspot](https://sbfspot.codeplex.com/) on a [Raspberry Pi](http://www.raspberrypi.org/) pushing data to <http://pvoutput.org>.

This guide also omits details that can be found on the web such as setting up the network, installing MySQL, etc. [Google it](http://lmgtfy.com)

## Requirements

 * Raspberry Pi 2
   * Bluetooth USB Stick
   * (Optional) Wifi USB Stick
 * or; Raspberry Pi 3
 * or; Raspberry Pi Zero W
 * http://pvoutput.org account with API key

## Steps

### Setup Raspberry Pi

First get the latest [Raspbian Lite](https://www.raspberrypi.org/downloads/raspbian/) image, as of writing this is Raspberian Stretch Lite April 2018.

Login to your pi so you can enable remote ssh access

 * Username: `pi`
 * Password: `raspberry`

First update your pi:

`sudo apt-get update && sudo apt-get upgrade`

Enable SSH and fix your timezone:

`sudo raspi-config`

> Network Options -> Hostname -> solarpi

> Interfacing Options -> SSH -> Enable

> Localisation Options -> Change Timezone

 * (_Optional_) [Create a new user and disable the default `pi` user](https://www.raspberrypi.org/documentation/linux/usage/users.md).
 * (_Optional_) [Setup networking](https://raspberrypi.stackexchange.com/questions/37920/how-do-i-set-up-networking-wifi-static-ip-address).

*_Windows_*: Use Git Bash as it contains useful unit tools, ie. ssh
*_Mac/Unix_*: Copy your ssh key to save typing passwords (from your local machine):

```shell
ssh-keygen -t rsa -C "your_email@example.com"
ssh-copy-id pi@solarpi.local
ssh pi@solarpi.local
```

### Find SMA Inverter

Setup your bluetooth:

`sudo apt-get install -y bluetooth`

Check your bluetooth and see if you can see the MAC address of your SMA inverter:

`hcitool scan`

### Build SBFSpot

Setup and install SBFspot to retrieve the data from the inverter and store it in a SQLite/MySQL database (this guide is using MySQL/Maria but should be very similar to setup sqlite).

`sudo apt-get install -y mysql-client libbluetooth-dev libboost-date-time-dev libboost-system-dev libboost-filesystem-dev libboost-regex-dev default-libmysqlclient-dev`

Create the following folders (following the [Filesystem Hierarchical Standards](http://www.pathname.com/fhs/)):

1. Source to `/usr/src`
```shell
sudo mkdir /usr/src/sbfspot.3
sudo chown pi:pi /usr/src/sbfspot.3
```
2. Data to `/var/opt`
```shell
sudo mkdir -p /var/opt/data/sbfspot.3
sudo chown pi:pi /var/opt/data/sbfspot.3
```
3. Logs to `/var/log`
```shell
sudo mkdir /var/log/sbfspot.3
sudo chown pi:pi /var/log/sbfspot.3
```

Download [SBFspot](https://github.com/SBFspot/SBFspot/releases) source code from the website, download and extract:

```shell
wget https://github.com/SBFspot/SBFspot/archive/V3.4.1.tar.gz
tar -xvf V3.4.1.tar.gz -C /usr/src/sbfspot.3/ --strip-components 1
```

Build the application:

```shell
cd /usr/src/sbfspot.3/SBFspot
make mysql
sudo make install_mysql
```

Will install to `/usr/local/bin/sbfspot.3/`

### Setup MySQL/MariaDB

Now setup the MySQL/MariaDB database, I will not go into how to install MySQL/MariaDB server so either install it locally or on another machine. If using another machine be sure to edit the `CreateMySQLUser.sql` file and change localhost to the IP address of your host, alternative use `'%'` for all IP addresses if security is not a concern.

Create the users/database in your MySQL/MariaDB server:

```shell
MYSQL_HOST=localhost
mysql -uroot -h$MYSQL_HOST -p < /usr/src/sbfspot.3/SBFspot/CreateMySQLDB.sql
mysql -uroot -h$MYSQL_HOST SBFspot -p < /usr/src/sbfspot.3/SBFspot/Update_340_MySQL.sql
mysql -uroot -h$MYSQL_HOST -p < /usr/src/sbfspot.3/SBFspot/CreateMySQLUser.sql
```

Test your MySQL/MariaSB connection:

```shell
mysql -uSBFspotUser -h$MYSQL_HOST -pSBFspotPassword SBFspot
```

### Setup SBFSpot Config

Edit your config (note if you screw it up just copy the template from `/usr/local/bin/sbfspot.3/SBFspot.default.cfg`):

```shell
cd /usr/local/bin/sbfspot.3
sudo cp SBFspot.default.cfg SBFspot.cfg
```

#### Manual

`sudo vi /usr/local/bin/sbfspot.3/SBFspot.cfg`

#### Semi-Automated

Set some environment variables (edit the variables and just paste into your shell):

```shell
BLUETOOTH_MAC_ADDRESS=00:00:00:00:00:00
PLANT_NAME=MyPlant
LATITUDE=50.80
LONGITUDE=4.33
MYSQL_HOST=localhost
ENABLE_CSV_REPORT=0
SBFSPOT_DATA_DIR=/var/opt/data/sbfspot.3
TZ=$(cat /etc/timezone)
```

The following sed commands will (in order):
 1. Update bluetooth address
 2. Update plant name (can contain spaces i believe)
 3. Update latitude
 4. Update longitude
 5. Uncomment MySQL section
 6. Update MySQL hostname
 7. Comment out SQLite
 8. Update CSV report flag (seems unnecessary due to DB)
 9. Update data directory
 10. Update timezone

```shell
sudo sed -i "s/00:00:00:00:00:00/${BLUETOOTH_MAC_ADDRESS}/" /usr/local/bin/sbfspot.3/SBFspot.cfg
sudo sed -i "s/MyPlant/${PLANT_NAME}/" /usr/local/bin/sbfspot.3/SBFspot.cfg
sudo sed -i "s/Latitude=.*/Latitude=${LATITUDE}/" /usr/local/bin/sbfspot.3/SBFspot.cfg
sudo sed -i "s/Longitude=.*/Longitude=${LONGITUDE}/" /usr/local/bin/sbfspot.3/SBFspot.cfg
sudo sed -i "s/^#SQL/SQL/" /usr/local/bin/sbfspot.3/SBFspot.cfg
sudo sed -i "s/SQL_Hostname=.*/SQL_Hostname=${MYSQL_HOST}/" /usr/local/bin/sbfspot.3/SBFspot.cfg
sudo sed -i "/SBFspot.db/ s/^/#/" /usr/local/bin/sbfspot.3/SBFspot.cfg
sudo sed -i "s/CSV_Export=1/CSV_Export=${ENABLE_CSV_REPORT}/" /usr/local/bin/sbfspot.3/SBFspot.cfg
sudo sed -i "s~/home/pi/smadata~${SBFSPOT_DATA_DIR}~" /usr/local/bin/sbfspot.3/SBFspot.cfg
sudo sed -i "s~Europe/Brussels~${TZ}~" /usr/local/bin/sbfspot.3/SBFspot.cfg
```

### Lets Rock!

Should be ready to rock, run the application in verbose mode and force the inquiry (as your lat/long may indicate its night time and it won't run):

`/usr/local/bin/sbfspot.3/SBFspot -v -finq`

Hopefully everything will be working as expected.

### Setup a Scheduled Task

Add a cron script to run every 5 minutes:

*Note*: This will destroy your current crontab.

```shell
echo '*/5 6-20 * * * /usr/local/bin/sbfspot.3/SBFspot -v > "/var/log/sbfspot.3/SBFspot$(date +\%Y\%m\%d).log" 2>&1
' | crontab
```

### Integrate pvoutput.org

The service/daemon reads the data from the database and sends it to Pvoutput.org

`sudo apt-get install libcurl4-openssl-dev`

```shell
cd /usr/src/sbfspot.3/SBFspotUploadDaemon
make mysql
sudo make install_mysql
```

### Setup SBFspotUpload Config

Edit your config (note if you screw it up just copy the template from `/usr/local/bin/sbfspot.3/SBFspotUpload.default.cfg`):

```shell
cd /usr/local/bin/sbfspot.3
sudo cp SBFspotUpload.default.cfg SBFspotUpload.cfg
```

#### Manual

`sudo vi /usr/local/bin/sbfspot.3/SBFspotUpload.cfg`

#### Semi-Automated

Set some environment variables (edit the variables and just paste into your shell):

```shell
PVOUTPUT_SID=SerialNmbrInverter_1:PVoutput_System_ID_1,SerialNmbrInverter_2:PVoutput_System_ID_2, etc
PVOUTPUT_API_KEY=acbd1234
MYSQL_HOST=localhost
SBFSPOT_LOG_DIR=/var/log/sbfspot.3
```

The following sed commands will (in order):
 1. Update pvoutput.org serial and SID
 2. Update pvoutput.org API key
 3. Uncomment MySQL section
 4. Update MySQL hostname
 5. Comment out SQLite
 6. Update log directory

```shell
sudo sed -i "s/PVoutput_SID=/PVoutput_SID=${PVOUTPUT_SID}/" /usr/local/bin/sbfspot.3/SBFspotUpload.cfg
sudo sed -i "s/PVoutput_Key=/PVoutput_Key=${PVOUTPUT_API_KEY}/" /usr/local/bin/sbfspot.3/SBFspotUpload.cfg
sudo sed -i "s/^#SQL/SQL/" /usr/local/bin/sbfspot.3/SBFspotUpload.cfg
sudo sed -i "s/SQL_Hostname=.*/SQL_Hostname=${MYSQL_HOST}/" /usr/local/bin/sbfspot.3/SBFspotUpload.cfg
sudo sed -i "/SBFspot.db/ s/^/#/" /usr/local/bin/sbfspot.3/SBFspotUpload.cfg
sudo sed -i "s~/home/pi/smadata/logs~${SBFSPOT_LOG_DIR}~" /usr/local/bin/sbfspot.3/SBFspotUpload.cfg
```

TODO: looks to automatically create a daemon/service
Add to systemd `/etc/systemd/system/SBFspotUploadDaemon.service`:

```shell
[Unit]
Description=Reads SMA inverter data from a database and sends it to Pvoutput.org
After=network.target

[Service]
Type=forking
User=pi
Restart=always
RestartSec=60
ExecStart=/usr/local/bin/sbfspot.3/SBFspotUploadDaemon -c /usr/local/bin/sbfspot.3/SBFspotUpload.cfg -p /run/SBFspotUploadDaemon.pid 2>&1> /dev/null
PIDFile=/run/SBFspotUploadDaemon.pid

[Install]
WantedBy=multi-user.target
Alias=SBFspotUploadDaemon.service
```

Enable the daemon on boot:

`sudo systemctl enable SBFspotUploadDaemon.service`

Start the daemon:

`sudo systemctl start SBFspotUploadDaemon.service`

Check the logs:

`cat /var/log/sbfspot.3/SBFspotUpload*`

### Automatically Purge Old Logs

This will remove log files older than 30 days:

```shell
(crontab -l ; echo '1 0 * * * find /var/log/sbfspot.3/ -name "SBFspot*.log" -mtime +30 -delete')| crontab -
```
