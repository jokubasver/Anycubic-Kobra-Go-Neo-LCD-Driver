# Anycubic Kobra Go/Neo Raspberry Pi LCD Driver

Running Klipper on your Anycubic Kobra Go or Neo? Make the stock LCD shine again with this custom LCD driver and KlipperScreen!

![Kobra Neo LCD KlipperScreen](/kobra-neo-lcd-1.jpg "Kobra Neo LCD KlipperScreen")

This is a fork of the excellent [fbcp-ili9341](https://github.com/juj/fbcp-ili9341) project, with a driver for the ST7796 SPI TFT display used in the Anycubic Kobra Go and Neo. 

SPI init sequence is based off the Anycubic Kobra Neo Marlin fork [BSP SPI TFT driver.](https://github.com/jokubasver/Kobra_Neo/blob/d3406176308f1839130edc08825f500a72c02f64/source/board/bsp_spi_tft.cpp#L406C12-L406C12)

Since the LCD used in the Kobra Go/Neo does not have touch, you can only use the display to view print stats. KlipperScreen does not currently support keyboard or rotary encoder control, and if it ever implements these features, then it would be possible to use the built-in rotary encoder for control. As it stands now - this is only for displaying printer info.

UPDATE: [SomeSpaceNerd](https://github.com/SomeSpaceNerd) has created a control interface for KlipperScreen using the rotary encoder as a mouse emulator. Check it out here: [KlipperScreen-Encoder-Driver](https://github.com/SomeSpaceNerd/KlipperScreen-Encoder-Driver)

Tested on a Raspberry Pi Zero 2W, but it should work on other Pi models, as long as they are supported by [fbcp-ili9341](https://github.com/juj/fbcp-ili9341)

### Hardware Setup
The LCD used in the Kobra Go/Neo uses an SPI interface, so we can directly connect it to the Raspberry Pi GPIO pins. All you need is some jumper wires.

The pinout is defined on the mainboard of the printer:

![Kobra-Go-Neo-Mainboard-LCD-Pinout-Silkscreen](https://github.com/jokubasver/Anycubic-Kobra-Go-Neo-LCD-Driver/assets/4341386/44d7e98c-0af0-4ce9-a5a3-752801bc9de5)

***However, the pinout on the silkscreen of the mainboard is not correct!*** This caused many headaches while building this project.

This is the correct pinout:

<img width="298" alt="Kobra-Go-Neo-LCD-2x5-IDC-Correct-Pinout-Mainboard-Side" src="https://github.com/jokubasver/Anycubic-Kobra-Go-Neo-LCD-Driver/assets/4341386/ae213679-0ccf-46e9-9dc2-2075c794b3d4">

Since KlipperScreen only accepts touch input, we can ignore the rotary encoder pins.

And so the wiring from the LCD to the Raspberry Pi GPIO is:
```
5V -> 5V
GND -> GND
SCK -> GPIO 11 (SPI SCLK)
MOSI -> GPIO 10 (SPI MOSI)
CS -> GPIO 8 (SPI CE0)
RESET -> GPIO 25
DC (Marked as MISO on the mainboard) -> GPIO 24
```

Now you need to decided where you will connect the jumpers wires to. 
 - You can remove the stock LCD cable entirely, and insert jumper wires directly to the connector of the LCD display.
 - Or, like I did, unplug the LCD cable from the mainboard of the printer, and insert jumper wires into the cable's connector:

![Kobra-Go-Neo-LCD-GPIO-To-LCD-Cable-2](https://github.com/jokubasver/Anycubic-Kobra-Go-Neo-LCD-Driver/assets/4341386/9e8cf3c4-8cdf-4fbc-bc14-174b968d43f0)

<img width="600" alt="Kobra-Go-Neo-LCD-GPIO-To-LCD-Cable" src="https://github.com/jokubasver/Anycubic-Kobra-Go-Neo-LCD-Driver/assets/4341386/77d8b5fd-2f6c-4b01-a3ee-9ab0b960040e">

With the hardware setup done, now you can proceed with the software installation.

### config.txt Setup
Before we start, we need to setup config.txt

In the terminal type `sudo nano /boot/config.txt`

Find `dtoverlay=vc4-kms-v4d` and add a `#` to the beginning of the line, so it should now be: `#dtoverlay=vc4-kms-v4d`

Scroll all the way to the bottom and add:
```bash
hdmi_group=2
hdmi_mode=87
hdmi_cvt=320 240 60 1 0 0 0
hdmi_force_hotplug=1
gpu_mem=160
avoid_warnings=2
```

This sets the resolution to that of the LCD display. Increasing the RAM might not be needed.

If your `/boot/config.txt` file has lines that look something like `dtoverlay=pitft28r, ...`, `dtoverlay=waveshare32b, ...` or `dtoverlay=flexfb, ...`, those should be removed.

The driver neither utilizes the default SPI driver, so a line such as `dtparam=spi=on` in `/boot/config.txt` should also be removed so that it will not cause conflicts.

Press CTRL+X, then Y to save.

To prevent the display going to sleep, type `sudo raspi-config`, navigate to `Display Options`, select `Screen Blanking` and press `No` to disable Screen Blanking. Now go back, save and reboot.


### Driver installation
Now that the config.txt is setup, we can clone the repo, setup cmake and compile the driver. 

Run in the console of your Raspberry Pi:

```bash
sudo apt update
sudo apt install cmake
cd ~
git clone https://github.com/jokubasver/Anycubic-Kobra-Go-Neo-LCD-Driver.git
cd Anycubic-Kobra-Go-Neo-LCD-Driver
mkdir build
cd build
cmake -DST7796=ON -DGPIO_TFT_DATA_CONTROL=24 -DGPIO_TFT_RESET_PIN=25 -DSPI_BUS_CLOCK_DIVISOR=30 -DSTATISTICS=0 -DUSE_DMA_TRANSFERS=ON ..
make -j
```
Now we can test if the driver and display works:
```bash
sudo ./fbcp-ili9341
```
If you see the terminal, congrats! You can move to the next step. If you get a white screen - check your wiring. If it still doesn't work, try uncommenting `dtoverlay=vc4-kms-v4d` in `/boot/config.txt` or change it to `dtoverlay=vc4-fkms-v4d`.

Now we need to make sure the driver starts at startup.
Edit the file `/etc/rc.local` by typing `sudo nano /etc/rc.local`, and add a line:

```bash
sudo /path/to/Anycubic-Kobra-Go-Neo-LCD-Driver/build/fbcp-ili9341 &
````
Make sure to change the path - for example, if your username is the default `pi`, then it should be: `sudo /home/pi/Anycubic-Kobra-Go-Neo-LCD-Driver/build/fbcp-ili9341 &`

Press CTRL+X, then Y to save, reboot by typing `sudo reboot now` and you should see the terminal after the Pi finishes booting.

At this point the driver installation is complete. Now it's time to setup KlipperScreen.

### KlipperScreen Setup

The easiest way to setup KlipperScreen is to use [KIAUH](https://github.com/dw-0/kiauh). Please follow the install instructions there to install KlipperScreen.

You will notice that after installing KlipperScreen, it will not show up on the LCD. We still need to do one more step in order for it to work.

Type `sudo nano /usr/share/X11/xorg.conf.d/99-fbturbo.conf` and you should see this:
```bash
Section "Device"
        Identifier      "Allwinner A10/A13 FBDEV"
        Driver          "fbturbo"
        Option          "fbdev" "/dev/fb0"

        Option          "SwapbuffersWait" "true"
EndSection
```

Change `Driver          "fbturbo"` to `Driver          "fbdev"`, press CTRL+X, then Y, and reboot: `sudo reboot now`

At this point you should see KlipperScreen on the LCD display.

**You are done!**

If you want to add a printer name and disable screen timeout, using your Klipper web interface (Mainsail/Fluidd) create a `KlipperScreen.conf` file in the config folder and add:
```bash
[main]
job_complete_timeout: 0
screen_blanking = off
use_dpms = False

[printer Anycubic Kobra Neo]
```
