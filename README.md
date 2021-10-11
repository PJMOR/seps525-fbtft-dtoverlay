# Device Tree Overlay for SEPS525 OLEDs using FBTFT on Raspberry Pi

This device tree overlay allows the use of SEPS525-based OLED displays such as the 160×128 1.69" and 1.45" display modules commonly found online. This overlay is primarily intended to be used on Raspberry Pis running Raspberry Pi OS, but can likely be adapted to work on other distros or even other boards. On versions of Raspberry Pi OS/Raspbian with a kernel older than 5.4, `fbtft_device` is still included, and can be used instead of this overlay as per instructions on [FBTFT's wiki.](https://github.com/notro/fbtft/wiki#step-by-step-using-fbtft-without-device-tree)

To use, the FBTFT driver for SEPS525 must be first enabled in the kernel. As of kernel 5.10.63 for Raspberry Pi OS the driver is included, but not enabled by default, so manually compiling the module is required.

### Downloading and installing kernel source

To download the source for the kernel currently in use, the [rpi-source](https://github.com/RPi-Distro/rpi-source) tool can be used:

Dependencies
```text
$ sudo apt update && sudo apt install -y git bc bison flex libssl-dev make libncurses5-dev
```

Install
```text
$ sudo wget https://raw.githubusercontent.com/RPi-Distro/rpi-source/master/rpi-source -O /usr/local/bin/rpi-source && sudo chmod +x /usr/local/bin/rpi-source && /usr/local/bin/rpi-source -q --tag-update

```
Run
```text
$ rpi-source
```

Alternatively, you can manually find the firmware and kernel hashes and download the kernel source using them:

```text
$ FIRMWARE_HASH=$(zgrep "* firmware as of" /usr/share/doc/raspberrypi-bootloader/changelog.Debian.gz | head -1 | awk '{ print $5 }')
$ KERNEL_HASH=$(wget https://raw.github.com/raspberrypi/firmware/$FIRMWARE_HASH/extra/git_hash -O -)
$ wget https://github.com/raspberrypi/linux/archive/$KERNEL_HASH.tar.gz
$ tar -xzf $KERNEL_HASH.tar.gz
```

### Enable module

From within the source's folder, run the `menuconfig` tool by:

```text
$ make menuconfig
```

Navigate to `Device Drivers`, then `Staging Drivers`, and finally `Support for small TFT LCD display modules`. Scroll down to `FB driver for the SEPS525 LCD Controller` and enable it. Save and exit.

Compile the module and copy it to the correct location:

```text
$ make modules_prepare -j6
$ ./scripts/diffconfig
$ make -C ~/linux M=drivers/staging/fbtft modules -j6
$ sudo cp ~/linux/drivers/staging/fbtft/fb_seps525.ko /lib/modules/$(uname -r)/kernel/drivers/staging/fbtft/
```

Enable SPI and disable overscan:

```text
$ sudo raspi-config nonint do_spi 0
$ sudo raspi-config nonint do_overscan 1
```

Enable FBTFT:

```text
$ echo "spi-bcm2835" | sudo tee /etc/modules-load.d/fbtft.conf
$ sudo depmod
```

### Enable overlay

Download `seps525-overlay.dts` from this repository. Within the same folder `seps525-overlay.dts` is located, compile the overlay:

```text
$ sudo dtc -@ -I dts -O dtb -o /boot/overlays/seps525.dtbo seps525-overlay.dts
```

To load the overlay on boot:

```text
$ echo "dtoverlay=seps525" | sudo tee -a /boot/config.txt
```

The line added to `config.txt` can be further appended to change certain settings in the format `dtoverlay=seps525,<param>=<val>`. Several parameters can be stringed together, separated by commas. Available parameters are:

- speed
- rotate
- txbuflen
- fps
- bgr
- debug

These correspond to the properties listed [on FBTFT's wiki](https://github.com/notro/fbtft/wiki/Device-Tree) with the same name.
