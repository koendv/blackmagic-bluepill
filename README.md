# Converting a Blue Pill to a Black Magic Probe

This document shows how to convert a STM32F103 Blue Pill to a Black Magic Probe gdb server. A Black Magic Probe (BMP) allows you to download firmware over USB, to set breakpoints, and inspect variables.

## Installing Firmware
Connect a STM32F103 Blue Pill for serial download:

* Set boot jumpers for boot from rom: Boot0=1, Boot1=0. 
* Connect a USB-Serial adapter with A9 to RX, A10 to TX. 
* Press reset.
* Upload the firmware: 
```
stm32flash -w  blackmagic_bluepill.hex /dev/ttyUSB0
``` 
with `/dev/ttyUSB0` the USB-serial adapter

* Set boot jumpers for boot from flash: Boot0=0, Boot1=0. 
* Press reset.
* Check the black magic probe shows up on usb: 
```
$ lsusb
Bus 001 Device 044: ID 1d50:6018 OpenMoko, Inc. Black Magic Debug Probe (Application)
```

## Creating Device Files

Download the [udev rules for the BMP](https://github.com/blacksphere/blackmagic/blob/master/driver/99-blackmagic.rules) and install them in 
 `/etc/udev/rules.d/99-blackmagic.rules`: 
 
```
$ wget https://github.com/blacksphere/blackmagic/raw/master/driver/99-blackmagic.rules
$ sudo cp 99-blackmagic.rules /etc/udev/rules.d/
$ sudo chown root:root /etc/udev/rules.d/99-blackmagic.rules 
$ sudo chmod 644 /etc/udev/rules.d/99-blackmagic.rules 
$ sudo udevadm control --reload-rules
```
Disconnect and re-connect the BMP. Check the device shows up in `/dev/`:
 
```
$ ls -l /dev/ttyBmp*
lrwxrwxrwx 1 root root 7 Mar 24 11:59 /dev/ttyBmpGdb -> ttyACM0
lrwxrwxrwx 1 root root 7 Mar 24 11:59 /dev/ttyBmpTarg -> ttyACM1
```
 
## Connecting to target

As target system we use another Blue Pill. Connect BMP and target like this:

Black Magic Probe  | Target  | Comment
--- | --- | ---
`GND` | `GND` |
`PB14` | `SWDIO` |
`PA5` | `SWCLK` |
`PA3` | `RXD` | Optional serial port
`PA2` | `TXD` | Optional serial port
`3V3` | `3V3` | Careful! Only connect one power source.

Connect the Black Magic Probe USB to the Raspberry.  Now we are ready to connect to the target system.
	
	$ arm-none-eabi-gdb -ex "target extended-remote /dev/ttyBmpGdb"
	...
	Remote debugging using /dev/ttyBmpGdb
	(gdb) monitor swdp
	Target voltage: unknown
	Available Targets:
	No. Att Driver
	 1      STM32F1 medium density M3/M4
	(gdb)

See the [overview of useful gdb commands](https://github.com/blacksphere/blackmagic/wiki/Useful-GDB-commands) for an  introduction to using the BMP.

## stm32duino

If you are using the Arduino IDE with the stm32duino add-on:

In the Arduino IDE, choose *Tools->Upload Method-> BMP (Black Magic Probe)* to upload firmware using the BMP.  This uploads the firmware using the `arm-none-eabi-gdb` command. The command line options to  `arm-none-eabi-gdb` are in the *tools.bmp_upload* section of `.arduino15/packages/STM32/hardware/stm32/1.8.0/platform.txt`.

The commands `stm32flash`, `dfu-util`, and `arm-none-eabi-gdb` can be found under `~/.arduino15`. ```find ~/.arduino15/ -name arm-none-eabi-gdb -print```

The rest of this document documents how to compile the `blackmagic_bluepill.hex` firmware.

## Compiling Firmware
The text assumes a linux with `stm32flash`, `dfu-util`,  `arm-none-eabi-gcc` and `arm-none-eabi-gdb` available.

Install [udev rules for the BMP](https://github.com/blacksphere/blackmagic/blob/master/driver/99-blackmagic.rules). Go to `~/.arduino15/packages/STM32/tools/STM32Tools/1.3.2/tools/linux/` and run `./install.sh`

Clone the BMP source git and compile.

	cd ~
	git clone https://github.com/blacksphere/blackmagic
	cd blackmagic/
	make PROBE_HOST=stlink
	cd src/
	
This creates two files, `blackmagic_dfu.bin` and `blackmagic.bin`.

	$ ls -l blackmagic_dfu.bin blackmagic.bin
	-rwxr-xr-x 1 koen koen 85068 Mar  9 19:16 blackmagic.bin
	-rwxr-xr-x 1 koen koen  7580 Mar  9 19:16 blackmagic_dfu.bin
	
Given the size of the firmware, we will need a Blue Pill with at least 128kbyte of flash.

First we install `blackmagic_dfu.bin`, the DFU bootloader. Connect the Blue Pill for serial download. Set boot jumpers for boot from rom: `Boot0`=1, `Boot1`=0. Connect a USB-Serial adapter with `A9` to `RX`, `A10` to `TX`. Press reset. Then:

	$ stm32flash -g 0x8000000 -w blackmagic_dfu.bin /dev/ttyUSB0
	stm32flash 0.5
	
	http://stm32flash.sourceforge.net/
	
	Using Parser : Raw BINARY
	Interface serial_posix: 57600 8E1
	Version      : 0x22
	Option 1     : 0x00
	Option 2     : 0x00
	Device ID    : 0x0410 (STM32F10xxx Medium-density)
	- RAM        : 20KiB  (512b reserved by bootloader)
	- Flash      : 128KiB (size first sector: 4x1024)
	- Option RAM : 16b
	- System RAM : 2KiB
	Write to memory
	Erasing memory
	Wrote address 0x08001d9c (100.00%) Done.
	
	Starting execution at address 0x08000000... done.
	
With the bootloader installed, set BMP boot jumpers to boot from flash: `Boot0`=0, `Boot1`=0. Disconnect the USB-Serial adapter. Connect the Blue Pill USB to the Raspberry.
	
	$ lsusb
	...
	Bus 001 Device 093: ID 1d50:6017 OpenMoko, Inc. Black Magic Debug Probe (DFU)
	
With the DFU bootloader running,  we download the blackmagic firmware using DFU.

	$ dfu-util -d 1d50:6018,:6017 -s 0x08002000:leave -D blackmagic.bin
	dfu-util 0.9
	
	Copyright 2005-2009 Weston Schmidt, Harald Welte and OpenMoko Inc.
	Copyright 2010-2016 Tormod Volden and Stefan Schmidt
	This program is Free Software and has ABSOLUTELY NO WARRANTY
	Please report bugs to http://sourceforge.net/p/dfu-util/tickets/
	
	dfu-util: Invalid DFU suffix signature
	dfu-util: A valid DFU suffix will be required in a future dfu-util release!!!
	Opening DFU capable USB device...
	ID 1d50:6017
	Run-time device DFU version 011a
	Claiming USB DFU Interface...
	Setting Alternate Setting #0 ...
	Determining device status: state = dfuIDLE, status = 0
	dfuIDLE, continuing
	DFU mode device DFU version 011a
	Device returned transfer size 1024
	DfuSe interface name: "Internal Flash   "
	Downloading to address = 0x08002000, size = 85068
	Download	[=========================] 100%        85068 bytes
	Download done.
	File downloaded successfully
	Transitioning to dfuMANIFEST state
	$ lsusb
	...
	Bus 001 Device 094: ID 1d50:6018 OpenMoko, Inc. Black Magic Debug Probe (Application)
	$ ls -l /dev/ttyBmp*
	lrwxrwxrwx 1 root root 7 Mar  9 19:53 /dev/ttyBmpGdb -> ttyACM0
	lrwxrwxrwx 1 root root 7 Mar  9 19:53 /dev/ttyBmpTarg -> ttyACM1

If the `/dev/ttyBmpGdb` and `/dev/ttyBmpTarg` devices do not show up, check  `/etc/udev/rules.d/99-blackmagic.rules` has been installed.

Make a backup copy of the firmware:
```
stm32flash -r blackmagic_bluepill.hex /dev/ttyUSB0 
```
## See also
[STM Discovery and Nucleo as Black Magic Probe](https://embdev.net/articles/STM_Discovery_and_Nucleo_as_Black_Magic_Probe#Building_Firmware_for_ST_Link_V2_Clones_and_Flash_Using_Two_Cheap_Clones)

*not truncated*	
