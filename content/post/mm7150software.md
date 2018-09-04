+++
date        = "2017-04-26T10:00:01+02:00"
draft       = false
title       = "Getting started with MM7150 and Raspberry Pi (software side)"
description = "Step-by-step tutorial to make the MM7150 imu works with the Raspberry Pi."
tags        = [ "MM7150", "Raspberry Pi", "Linux" ]
topics      = [ "Development"]
author      = "Christian-Nils Boda"
+++

# Acknowledgements
First of all, I would like to thank Benjamin Tissoires, for his huge help on getting the MM7150 working with my Raspberry Pi. It was the first time for me to go in the Linux kernel and his help was very valuable especially that he is the original author of the i2c-hid driver.

# Introduction
In this guide, you will find everything you need, software-side, to make your MM7150 work with your Raspberry Pi version 3 (it may work with previous versions). I am using the kernel 4.4+ of Raspbian. I tried the latest kernel (4.9+) and it does not work. So please check if your version is the right one. In this guide, we will assume that you are starting to configure your OS from scratch. Also, I will only show how to do it with a headless Raspberry Pi, using the SSH connection. To simplify the guide and the installation, I am working exclusively with my root session. If you decide to do not do the same, remember to add `sudo` where administrative rights are required.

# Installation of Raspbian
The first step is to install Raspbian Lite on the SD card, see [the offical guide][0].

# Connect to the headless Raspberry Pi

If you are using a ssh connection to control the Raspberry Pi, you will need to add an empty file named 'ssh' in the boot partition of the sd card (it will enable the ssh connection, see [1]).
After booting up the Raspberry Pi, you start the ssh session:

```
ssh pi@192.168.1.10
```
You should change the ip address for your Raspberry Pi's ip address. Reminder: the default password is *raspberry*. After logging, you will mostly work as administrator, so type:
```
sudo -i
```
This is only optional, if you decide to do not do so, you will have to add `sudo` before most of the commands you will type.

# Update
After the first login, you should update your raspberry pi by typing:

```
apt-get update
apt-get upgrade
```

# (Optional) Raspi-config
You can change the password and enlarge filesystem using the famous command `raspi-config`.

# Modification of i2c-hid.c to disable the Power Management
The MM7150 module is an amazing IoT chip, it can be put to sleep and consume nearly no energy. Microchip does it by using a wake-up line. Unfortunately, this is not a standard way and it is not implemented in the driver `i2c-over-hid`. My limited skills do not allow me to add this wake-up line handling in the driver, therefore I simply disable the power management. That means that the module will not be put to sleep.
If anyone has the skills, please do it, and let me know! :)

Anyway, here is what you need to change in `linux/drivers/hid/i2c-hid.c`. You need to change the following line
```
SET_RUNTIME_PM_OPS(i2c_hid_runtime_suspend, i2c_hid_runtime_resume,NULL)
```
with
```
SET_RUNTIME_PM_OPS(NULL,NULL,NULL)
```

# Modification of hid-sensor-hub.c
One simple but really important step for the 4.4+ kernel is to add the MM7150 quirk (see [patch][7] on the kernel v.4.9). You can simply replace your `linux/drivers/hid/hid-sensor-hub.c` file with the one on the [offical repository][6].

# Compile kernel
In order to compile the kernel properly, you should follow the instructions described on the [official raspbian tutorial][2]. But, here are the main steps for this compilation.

* Install the dependencies
* Clone linux repository 

    ```
    git clone -b rpi-4.4.y --depth=1 https://github.com/raspberrypi/linux
    ```
* Configure the kernel (read instructions on [this page][3] and see the next section for the lists of drivers to enable in the kernel)
    ```
    make bcm2709_defconfig
    ```
* Compile the source files

    ```
    make -j4 zImage modules dtbs
    ```
* Install modules, compiled overlays and the image of the kernel

    ```
    make modules_install
    cp arch/arm/boot/dts/*.dtb /boot/
    cp arch/arm/boot/dts/overlays/*.dtb* /boot/overlays/
    cp arch/arm/boot/dts/overlays/README /boot/overlays/
    cp arch/arm/boot/zImage /boot/KERNEL7.img
    ```

# Configure kernel drivers

After installing the ncurses development headers:

```
apt-get install libncurses5-dev
```

You can access the configuration menu by typing:

```
make menuconfig
```

You will have to browse the menu in order to activate the following:

```
# Device Drivers > HID Support > I2C HID Support > HID over I2C transport layer
# Device Drivers > HID Support > Special HID Drivers > HID Sensors framework support
# Device Drivers > HID Support > Special HID Drivers > HID Sensors hub custom sensor support
# Device Drivers > Industrial I/O support
# Device Drivers > Industrial I/O support > Accelerometers > HID Accelerometers 3D
# Device Drivers > Industrial I/O support > Digital gyroscope sensors > HID Gyroscope 3D 
# Device Drivers > Industrial I/O support > Magnetometer sensors > HID Magenetometer 3D
# Device Drivers > Industrial I/O support > Inclinometer sensors > HID Inclinometer 3D 
# Device Drivers > Industrial I/O support > Inclinometer sensors > HID Device Rotation
# Industrial I/O Support
# All trigger references
# Enable software triggers support
# Triggers - standalone
```

I included all of them in the kernel (by typing `Y`). You can modularize the features if you want, but, for now, it is easier to include them by default.

# Create the overlay file

Below you will find my overlay for MM7150 (the same kind of overlay can be used for other device using i2c-hid technology). You can find more detailed infos on [this page][4].

Place the code in a file named `mm7150-overlay.dts` in `~/linux/arch/arm/boot/dts/overlays`

```
// Definitions for HID over I2C
/dts-v1/;
/plugin/;

/{
   compatible = "brcm,bcm2835", "brcm,bcm2708", "brcm,bcm2709";

   fragment@0 {
      target = <&i2c_arm>;
      __overlay__ {
         status = "okay";
      };
   };

   fragment@1 {
      target = <&gpio>;
      __overlay__ {
         i2c_hid_pins: i2c_hid_pins {
            brcm,pins = <5>;
            brcm,function = <0>;
         };
      };
   };

   fragment@2 {
      target = <&i2c_arm>;
      __overlay__{
         #address-cells = <1>;
         #size-cells = <0>;

         i2c_hid: i2c-hid-dev@40 {
            compatible = "hid-over-i2c";
            reg = <0x40>;
            hid-descr-addr = <0x0001>;
            interrupt-parent = <&gpio>;
            interrupts = <5 0x8>;
         };

      };
   };

   __overrides__ {
      gpiopin = <&i2c_hid_pins>, "brcm,pins:0",
               <&i2c_hid>, "interrupts:0";
      addr = <&i2c_hid>, "reg:0";
      };

};
```

I am not an expert, but here are some simple explanations on how the overlay works. The fragment@2 declares that the target is the i2c port and that the hardware at the registery <0x40> should use the driver "hid-over-i2c". The interrupt is configured as well, by default the pin 5 is used. It can be overriden by changing the parameter *gpiopin* in config.txt. We use the pin 5 here, so we do not need to do so.

# Compile the overlay
Before being able to enable the overlay, you will need to compile it. First, you have to install the device tree compiler.

```
apt-get install device-tree-compiler
```
Then you place yourself in `~/linux/arch/arm/boot/dts/overlays` and you can compile your dts file by typing:

```
dtc -@ -I dts -O dtb -o mm7150.dtbo mm7150-overlay.dts
```
Move the compiled file to the boot folder:
```
cp arch/arm/boot/dts/overlays/*.dtb* /boot/overlays/
```

# Enable i2c_arm
Add the line `dtparam=i2c_arm=on,i2c_arm_baudrate=400000` in `/boot/config.txt` to enable i2c and to set the baudrate to 400kbps.
Add the line `i2c-dev` in `/etc/modules` if you did not include i2c-dev in the kernel but only modularized it.

# Enable the MM7150's overlay
If you want to enable the overlay at boot, you must add the following line in your `config.txt` file.

```
dtoverlay=mm7150
```

Potentially, you can add, after this line, the parameters if you need to change the default values of the register address or the interrupt line pin. You can look up on [this page][5] to get the proper pin number, BCM number is the one you want here. More information on how to configure config.txt can be read on [this page][4]. Additionally, you can read the file which is located in `arch/arm/boot/dts/overlays/README` to know which parameters are available.

# Misc commands

## To check is the device is recognized as a I2C device
You have to install i2c-tools first, in order to use the command `i2cdetect`.
```
install i2c-tools
i2cdetect -y 1
```
## To debug the i2c-hid driver
You must add `i2c_hid.debug=1` in your cmdline.txt file in order to see the debug messages (`dmesg | grep i2c`).

## To add/remove the overlay in "real time"
Type `dtoverlay -h` if you want more help. I used primarily the following commands:
```
dtoverlay -r mm7150 # to remove the overlay called mm7150
dtoverlay mm7150 # to add the overlay called mm7150
dtoverlay mm7150 gpiopin=3 # to add the overlay called mm7150 with the dtparam=gpiopin=3 (you attribute the interrupt line to GPIO 3 instead of 5)
```

## To access values from the raw sensors
See hid-sensor-custom [8]

## To see description of IIO devices features
See [https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/ABI/testing/sysfs-bus-iio?id=HEAD][9]

# Communicate with MM7150
See next post (to be written) about how to communicate with the module using IIO commands.

[0]: https://www.raspberrypi.org/documentation/installation/installing-images/README.md
[1]: https://www.raspberrypi.org/documentation/remote-access/ssh/
[2]: https://www.raspberrypi.org/documentation/linux/kernel/building.md
[3]: https://www.raspberrypi.org/documentation/linux/kernel/configuring.md
[4]: https://www.raspberrypi.org/documentation/configuration/device-tree.md
[5]: https://pinout.xyz/
[6]: https://github.com/torvalds/linux/blob/master/drivers/hid/hid-sensor-hub.c
[7]: https://github.com/torvalds/linux/commit/5cc5084dd9afa2f9bf953b0217bdb1b7c2158be1
[8]: https://www.kernel.org/doc/Documentation/hid/hid-sensor.txt
[9]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/ABI/testing/sysfs-bus-iio?id=HEAD