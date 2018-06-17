# Overview

This document will outline a shell script to enable GPIO on a [Raspberry Pi](http://www.raspberrypi.org/). Mainly so I dont forget about the settle command.

## Output

```shell
echo 12 > /sys/class/gpio/export
udevadm settle
echo out > /sys/class/gpio/gpio12/direction
```

Turn on and off:

> `echo 0 > /sys/class/gpio/gpio12/value`

> `echo 1 > /sys/class/gpio/gpio12/value`

## Input

```shell
echo 7 > /sys/class/gpio/export
udevadm settle
echo in > /sys/class/gpio/gpio7/direction
```

Check value:

>`cat /sys/class/gpio/gpio7/value`

## Script

Simple shell script to enable all PINs:

```shell
#!/bin/bash

# Input PINs
for i in {7,8}
do
    echo $i > /sys/class/gpio/export
    udevadm settle
    echo in > /sys/class/gpio/gpio$i/direction
done

# Output PINs
for i in {12,16,20,21}
do
    echo $i > /sys/class/gpio/export
    udevadm settle
    # Default high - https://www.kernel.org/doc/Documentation/gpio/sysfs.txt
    echo high > /sys/class/gpio/gpio$i/direction
done
```