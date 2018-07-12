# Overview

This document will outline the steps on how to setup your [Raspberry Pi](http://www.raspberrypi.org/) to control your garage door.

## Requirements

 * Raspberry Pi 2
   * (Optional) Wifi USB Stick
 * or; Raspberry Pi 3
 * or; Raspberry Pi Zero W with PIN out header
 * [5V 4 Channel Relay](https://core-electronics.com.au/5v-4-channel-relay-module-10a.html)
 * [Magnetic Reed Switch](https://www.jaycar.com.au/magnetic-reed-switch-module/p/XC4476) (_Optional_)

## Steps

### Setup Raspberry Pi

First get the latest [Raspbian Lite](https://www.raspberrypi.org/downloads/raspbian/) image.

Login to your pi so you can enable remote ssh access

 * Username: `pi`
 * Password: `raspberry`

First update your pi:

`sudo apt-get update && sudo apt-get upgrade`

Enable SSH and fix your timezone:

`sudo raspi-config`

> Network Options -> Hostname -> garagepi

> Interfacing Options -> SSH -> Enable

> Localisation Options -> Change Timezone

 * (_Optional_) [Create a new user and disable the default `pi` user](https://www.raspberrypi.org/documentation/linux/usage/users.md).
 * (_Optional_) [Setup networking](https://raspberrypi.stackexchange.com/questions/37920/how-do-i-set-up-networking-wifi-static-ip-address).

*_Mac/Unix_*: Copy your ssh key to save typing passwords (from your local machine):

```shell
ssh-keygen -t rsa -C "your_email@example.com"
ssh-copy-id pi@garagepi.local
ssh pi@garagepi.local
```

*_Windows_*: Use Git Bash as it contains useful unit tools, ie. ssh

### Setup simple REST server

Add python3 and pip:

`sudo apt-get install python3-pip`

Install [Flask](http://flask.pocoo.org/) and [GPIO](https://pypi.org/project/RPi.GPIO/) libraries:

`sudo pip3 install Flask RPi.GPIO`

Create a simple python REST JSON server script, ie. garagepi.py:

```python
import RPi.GPIO as GPIO
import time
from werkzeug.exceptions import HTTPException
from flask import Flask, Blueprint, jsonify

errors = Blueprint('errors', __name__)

# Serialise all errors as JSON
@errors.app_errorhandler(Exception)
def handle_error(error):
    status_code = error.code if isinstance(error, HTTPException) else 500
    success = False
    response = {
        'success': success,
        'error': {
            'type': error.__class__.__name__,
            'message': str(error)
        }
    }
    return jsonify(response), status_code

app = Flask(__name__)
app.register_blueprint(errors)

GPIO.setmode(GPIO.BCM)
GPIO.setup([7,8], GPIO.IN)
GPIO.setup([12,16,20,21], GPIO.OUT, initial=GPIO.HIGH)

@app.route('/gpio/toggle/<int:gpio>', methods=['POST'])
def toggle_gpio(gpio):
    try:
        GPIO.output(gpio, GPIO.LOW)
        time.sleep(0.5)
        GPIO.output(gpio, GPIO.HIGH)
        success = True
    except Exception:
        success = False
    return jsonify({"success": success, "gpio": gpio})

@app.route('/door/state', methods=['GET'])
def door_state():
    if GPIO.input(7):
        is_open = False
    else:
        is_open = True
    return jsonify({"success": True, "open": is_open})

@app.route('/door/open', methods=['POST'])
def door_open():
    return toggle_gpio(20)

@app.route('/door/stop', methods=['POST'])
def door_stop():
    return toggle_gpio(16)

@app.route('/door/close', methods=['POST'])
def door_close():
    return toggle_gpio(12)
```

### Daemonize

Install [Gunicorn](http://gunicorn.org/) WSGI server:

`sudo pip3 install gunicorn`

(_Optional_) If you have issues starting Gunicorn, try:

`sudo pip3 install --upgrade setuptools`

Add `/etc/systemd/system/garagepi.service`:

```shell
[Unit]
Description=GaragePi
After=network.target

[Service]
PIDFile=/run/garagepi/garagepi.pid
User=pi
Group=pi
RuntimeDirectory=garagepi
WorkingDirectory=/opt/garagepi
ExecStart=/usr/local/bin/gunicorn --bind 0.0.0.0:8000 --pid /run/garagepi/garagepi.pid garagepi:app
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s TERM $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

Enable on boot `sudo systemctl enable garagepi.service`

Start `sudo systemctl start garagepi.service`

### Test

>`curl -X GET http://garagepi.local:8000/door/state`

>`curl -X POST http://garagepi.local:8000/door/open`

>`curl -X POST http://garagepi.local:8000/door/stop`

>`curl -X POST http://garagepi.local:8000/door/close`

