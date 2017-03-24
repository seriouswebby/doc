# Overview

This document will outline the steps on how to install [SBFspot](https://sbfspot.codeplex.com/) on a [Raspberry Pi](http://www.raspberrypi.org/) pushing data to <http://pvoutput.org>.

This guide also omits details that can be found on the web such as setting up the network, installing MySQL, etc. [Google it](http://lmgtfy.com)

## Requirements

 * Raspberry Pi 2
   * Bluetooth USB Stick
   * (Optional) Wifi USB Stick
 * or; Raspberry Pi 3
 * http://pvoutput.org account with API key

## Steps

### Setup Raspberry Pi

First get the latest [Raspbian Jessie Lite](https://www.raspberrypi.org/downloads/raspbian/) image, current at time of writing is 2017-03-02.

Login to your pi so you can enable remote ssh access

 * Username: `pi`
 * Password: `raspberry`

First update your pi:

`sudo apt-get update && sudo apt-get upgrade`

Enable SSH and fix your timezone:

`sudo raspi-config`

> Interfacing Options -> SSH -> Enable

> Localisation Options -> Change Timezone

 * (_Optional_) [Create a new user and disable the default `pi` user](https://www.raspberrypi.org/documentation/linux/usage/users.md).
 * (_Optional_) [Setup a static IP](https://www.modmypi.com/blog/how-to-give-your-raspberry-pi-a-static-ip-address-update).

*_Mac/Unix_*: Copy your ssh key to save typing passwords (from your local machine):

```shell
ssh-keygen -t rsa -C "your_email@example.com"
ssh-copy-id pi@192.168.1.101
ssh pi@192.168.1.101
```

*_Windows_*: Use Putty/Pageant

### Find SMA Inverter

Setup your bluetooth:

`sudo apt-get install bluetooth`

Check your bluetooth and see if you can see the MAC address of your SMA inverter:

`hcitool scan`

### Build SBFSpot

Setup and install SBFspot to retrieve the data from the inverter and store it in a SQLite/MySQL database (this guide is using MySQL but should be very similar to setup sqlite).

`sudo apt-get install mysql-client-5.5 libbluetooth-dev libboost-all-dev libmysqlclient-dev`

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

Download [SBFspot](https://sbfspot.codeplex.com/) source code from the website, copy it across and extract:

```shell
scp SBFspot_SRC_331_Linux_Win32.tar.gz pi@192.168.1.101:.
tar -xvf SBFspot_SRC_331_Linux_Win32.tar.gz -C /usr/src/sbfspot.3/
```

Build the application:

```shell
cd /usr/src/sbfspot.3/SBFspot
sudo make install_mysql
```

Will install to `/usr/local/bin/sbfspot.3/`

### Setup MySQL/MariaDB

Now setup the MySQL/MariaDB database, I will not go into how to install MySQL/MariaDB server so either install it locally or on another machine.

Create the users/database in your MySQL/MariaDB server:

```shell
mysql -u root -h localhost -p < /usr/src/sbfspot.3/SBFspot/CreateMySQLDB.sql
mysql -u root -h localhost -p < /usr/src/sbfspot.3/SBFspot/CreateMySQLUser.sql
```

### Setup SBFSpot Config

Edit you config (note if you screw it up just copy the template from `/usr/src/sbfspot.3/SBFspot/SBFspot.cfg`):

#### Manual

`sudo vi /usr/local/bin/sbfspot.3/SBFspot.cfg`

#### Automated

Set some environment variables (edit the variables and just paste into your shell):

```shell
BLUETOOTH_MAC_ADDRESS=00:00:00:00:00:00
PLANT_NAME=MyPlant
LATITUDE=50.80
LONGITUDE=4.33
MYSQL_IP_ADDRESS=localhost
ENABLE_CSV_REPORT=0
SBFSPOT_DATA_DIR=/var/opt/data/sbfspot.3
TZ=$(cat /etc/timezone)
```

The following sed commands will (in order):
 1. Updating bluetooth address
 2. Updating plant name (can contain spaces i believe)
 3. Updating latitude
 4. Updating longitude
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
sudo sed -i "s/SQL_Hostname=.*/SQL_Hostname=${MYSQL_IP_ADDRESS}/" /usr/local/bin/sbfspot.3/SBFspot.cfg
sudo sed -i "/SBFspot.db/ s/^/#/" /usr/local/bin/sbfspot.3/SBFspot.cfg
sudo sed -i "s/CSV_Export=1/CSV_Export=${ENABLE_CSV_REPORT}/" /usr/local/bin/sbfspot.3/SBFspot.cfg
sudo sed -i "s~/home/pi/smadata~${SBFSPOT_DATA_DIR}~" /usr/local/bin/sbfspot.3/SBFspot.cfg
sudo sed -i "s~Europe/Brussels~${TZ}~" /usr/local/bin/sbfspot.3/SBFspot.cfg
```

### Lets Rock!

Should be ready to rock, run the application in verbose mode and force the inquiry (as your lat/long may indicate its night time and it won't run):

`/usr/local/bin/sbfspot.3/SBFspot -v -finq`

TODO:  The service/daemon reads the data from the database and sends it to Pvoutput.org

