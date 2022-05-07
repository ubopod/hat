# Utilities to create, flash and dump HAT EEPROM images.

There is a complete, worked example at:
https://www.raspberrypi.org/forums/viewtopic.php?t=108134

## Overview

1. Tools available:
	* `eepmake`: Parses EEPROM text file and creates binary `.eep` file
	* `eepdump`: Dumps a binary .eep file as human readable text (for debug)
	* `eepflash.sh`: Write or read .eep binary image to/from HAT EEPROM

2. Edit `eeprom_setting.txt` to suit your specific HAT.

3. Run `./eepmake eeprom_settings.txt eeprom.eep` to create the eep binary

#### Getting Started on Raspberry Pi

0. Create tools with `make && sudo make install` (If you did the previous steps on your Pi, you don't need to do this).

First of all, you need to setup EEPROM utils (to make EEPROM content and flash it) and device-tree compiler (dtc).
For the device tree, I followed advice from Adafruit:https://learn.adafruit.com/introduction-to-the-beaglebone-black-device-tree/compiling-an-overlay

```
git clone https://github.com/raspberrypi/hats.git
wget -c https://raw.githubusercontent.com/RobertCNelson/tools/master/pkgs/dtc.sh
chmod +x dtc.sh
./dtc.sh
```
then go to the following directory:

```
cd hats/eepromutils/
```
and run:

```make && sudo make install```

This should make `eepdump`, `eepmake` executables.

1. Configure `boot/config.txt`

add this line `dtparam=i2c_vc=on` to `boot/config.txt` and reboot. After reboot 

```

2. Disable EEPROM write protection
	* On Ubo pod the EEPROM write protect is connected to GPIO 16. To disable write protect, it must be pulled low:
	```
	raspi-gpio set 16 op
	raspi-gpio set 16 dl
	```
	On this platforms:
	* Sometimes this requires a jumper on the board
	* Check your schematics
3. Make sure you can talk to the EEPROM
	* In the HAT specification, the HAT EEPROM is connected to pins that can be driven by I2C0.
	After reboot, confirm if EEPROM chip is recognized on bus 0 and on address 0x50:

	'i2cdetect -y 0' 

	```
	     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
	00:                         -- -- -- -- -- -- -- -- 
	10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
	20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
	30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
	40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
	50: 50 -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
	60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
	70: -- -- -- -- -- -- -- --   
	```
	Normally, you can skip this step, and assume things are working.

4. Modify and customize EEPROM settings file and create binary:

You need to modify eeprom_settings.txt to create your own version of HAT board. You don't have to modify UUID, as it will be auto-generated.

Then, you can create an eep file, based on you eeprom_settings.txt file. Basically, an eep file is a binary version of this file, which is ready to flash on EEPROM.

```
./eepmake eeprom_settings.txt myhat.eep
```

5 . Flash zeros on EEPROM to erase existing content:

You can now write the eep file you created in step 4 on EEPROM. If you have followed the design guide, you have a 24c32 memory (4k). But your myhat.eep file is probably smaller than 4k. As you don't know the current state of your EEPROM, you may have conflict, as your myhat.eep could be misread. To avoid that, we shall start by cleaning EEPROM.

Use this dd command to generate a 4k file, padded with zero (an excellent choice, zeros are my favorites!). If you have another EEPROM size, just change count value according to your real EEPROM size.

```
dd if=/dev/zero ibs=1k count=4 of=blank.eep
```

To be sure, you can review this binary, using hexdump :

```
> hexdump blank.eep

0000000 0000 0000 0000 0000 0000 0000 0000 0000
*
0001000
```

Next, you can now upload `blank.eep` to your EEPROM :

```
sudo ./eepflash.sh -w -f=blank.eep -t=24c32
```

make sure you have the write protect pull down to zero, otherwise you will get I/O error. To make sure the contect was correctly written, you can read back from EEPROM using the following command:

```
sudo ./eepflash.sh -r -f=blank_readback.eep -t=24c32
```

and then, do hexdump to make sure the content is empty:

```
hexdump blank_readback.eep
```

5. Flash eep file 

Then, you can upload your own myhat.eep.

`sudo ./eepflash.sh -w -t=24c32 -f=eeprom.eep`

to confirm, read back the content from EEPROM:

`sudo ./eepflash.sh -r -t=24c32 -f=eeprom_readback.eep`

you can then use the eepdump command to convert the eep file to text and compare it with `eeprom_setting.txt` file.

```
./eepdump eeprom_readback.eep eeprom_setting_readback.txt
```

If the contents match, you can reboot your Raspberry Pi.


5. Enable EEPROM write protection, by undoing step 1:

this can be done by bringing back GPIO value to 1:

```
raspi-gpio set 16 dh
```

6. Verify device tree 

To check if your HAT is recognized, just go to /proc/device-tree/. If you see a hat directory, you are a winner:)


```
cd /proc/device-tree/hat/
more vendor
more product
```

The next step is to allow autoconfiguration of this HAT, following device-tree usage. In our example, we have a led connected to GPIO 18 (pin 12). I suggest you to test it before, using /sys/class/gpio :

```
sudo sh -c 'echo "18" > /sys/class/gpio/export'
sudo sh -c 'echo "out" > /sys/class/gpio/gpio18/direction'
sudo sh -c 'echo "1" > /sys/class/gpio/gpio18/value'
```


