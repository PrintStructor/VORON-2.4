If you want to use Klipperscreen on an Amazon device there are some additional steps to do, which are:

**Android KlipperScreen now with newer Version possible for Amazon Tablets, ...!!**

First of all, you need the Google Playstore installed on your device.
I'm doing this with the app "Fire Toolbox" just google it and follow the instructions. 

Here you can find the attached instructions in original as well:
https://github.com/naruhaxor/AndroidKlipperScreen?fbclid=IwAR0NORYEs5xTHf0k-NLsJ_1e6LSrRYANj4CpA0nnWNqL1Q9bj50MmPtXAoU

# AndroidKlipperScreen
a simple writeup /installer script to use android Devices and a UI for KlipperScreen for RaspberryPI
Thanks to JHS on the Klipper Forums for most of his code

How to Use Android Device as a KlipperScreen UI

**Step 1** 
Download and install Klipper Screen Via Kiuah https://github.com/th33xitus/kiauh

**Step 2**
Download and install XSDL and Configure.

  Android can use the Play Store https://play.google.com/store/apps/details?id=x.org.server&hl=en_US&gl=US
  
  Fire Devices need to Sideload APK called x.org.server.apk listed in the data folder (uploading shortly however here is a direct download link) This version IS required for Fire devices
  
  https://sourceforge.net/projects/libsdl-android/files/apk/XServer-XSDL/XServer-XSDL-1.11.40.apk/download

Side Note:
On newer Amazon devices this app won't work any more and you get into a bootloop of the app. Follow my instructions on the end of this file!!

  After Install open app Youll see SDL splash screen click “CHANGE DEVICE CONFIGURATION” 
Click Mouse Emulation the Mouse Emulation Mode then Select Desktop, No Emulation


**Step 3** 
Setup ADB on your PI
```
sudo apt-get update
sudo apt-get install android-tools-adb
```
Ensure device has Android Debugging enabled
For reference look up your device and how to enable android debugging

connect android device via USB to PI

**Step 4**
Time to create the loading script to forward the display to XSDL im doing this in putty to ssh to my PI

SSH into PI then move to KlipperScreen folder
```
cd ~/KlipperScreen
nano ./launch_klipperscreen.sh
```
paste this code into your nano screen


```
#!/bin/bash
# forward local display :100 to remote display :0
adb forward tcp:6100 tcp:6000

adb shell dumpsys nfc | grep 'mScreenState=' | grep OFF_LOCKED > /dev/null 2>&1
if [ $? -lt 1 ]
then
    echo "Screen is OFF and Locked. Turning screen on..."
    adb shell input keyevent 26
fi

adb shell dumpsys nfc | grep 'mScreenState=' | grep ON_LOCKED> /dev/null 2>&1
if [ $? -lt 1 ]
then
    echo "Screen is Locked. Unlocking..."
    adb shell input keyevent 82
fi

# start xsdl
adb shell am start-activity x.org.server/.MainActivity

ret=1
timeout=0
echo -n "Waiting for x-server to be ready "
while [ $ret -gt 0 ] && [ $timeout -lt 60 ]
do
    xset -display :100 -q > /dev/null 2>&1
    ret=$?
    timeout=$( expr $timeout + 1 )
    echo -n "." 
    sleep 1
done
echo ""
if [ $timeout -lt 60 ]
then
    DISPLAY=:100 /home/pi/.KlipperScreen-env/bin/python3 /home/pi/KlipperScreen/screen.py
    exit 0
else
    exit 1
fi
```
Save Buffer by "Ctrl X" then yes
now run
```
sudo chmod +x ./launch_klipperscreen.sh
```
this is nessessary to make our script executable

**Step 5**
test the screen output by running the script
```
./launch_klipperscreen.sh
```
this is to verify a few items 
1. that the screen actually works 
2. to determine whether or not KlipperScreen is connecting to the printer

If your screen works and connects to the printer
AWESOME move onto step 6 there is nothing further for you to do

if your screen works and does not connect to the printer we need to retrieve the API Key
- this is normally caused by logins being enabled on the Fluidd UI this is an easy hurdle however
- retrieve your API key and place in a Configuration file
    
    
    example code Save in klipper_config (/home/pi/klipper_config/KlipperScreen.conf)
    ```
    # Define printer and name. Name is anything after the first printer word
    [printer yourprintername]  
    # Define the moonraker host/port if different from 127.0.0.1 and 7125
    moonraker_host: 127.0.0.1
    moonraker_port: 7125
    # Moonraker API key if this is not connecting from a trusted client IP
    moonraker_api_key: 
    #~# --- Do not edit below this line. This section is auto generated --- #~#
    
    #~#
    #~# [main]
    #~# screen_blanking = off
    #~# theme = material-dark
    #~# confirm_estop = True
    #~#
    ```
    now kill the script by pressing Ctrl C twice
    
**Step 6**
Now its time to setup the Script as a system service that survives throughout reboots
look at example.service in data folder that is word for work what you need in your file i have it listed as i am going to be compiling an autoscript

time to go to the Systemd
```
cd /etc/systemd/system
```
now time to create file
```
sudo nano ./KlippyScreenAndroid.service
```
paste in nano
```
[Unit]
Description=KlippyScreen
After=moonraker.service
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=1
User=pi
WorkingDirectory=/home/pi/KlipperScreen
ExecStart=/usr/bin/bash  /home/pi/KlipperScreen/launch_klipperscreen.sh

[Install]
WantedBy=multi-user.target
```
save, now time to enable and reload services
```
sudo systemctl daemon-reload
sudo systemctl enable KlippyScreenAndroid
sudo systemctl restart KlippyScreenAndroid
```
Now it should be working and no matter what it will run every reboot

Side Note to get working over Wi-Fi instead of USB ADB

Follow all steps Skip 3 and put this code in script instead under Step 4
```
#!/bin/bash
DISPLAY=(your ip from blue screen):0 /home/pi/.KlipperScreen-env/bin/python3 /home/pi/KlipperScreen/screen.py
```
With the Wi-Fi method you can create multiple scripts to push to separate devices however if you lose connection you'll have to restart the service to push to the devices again

a simple writeup /installer script to use android Devices and a UI for KlipperScreen for RaspberryPI Thanks to JHS on the Klipper Forums for most of his code
How to Use Android Device as a KlipperScreen UI

**Newer Amazon Tablets or XSDL Version beyond 1.11.40**
If Klipperscreen does not start because a newer version of Xserver XSDL is used and / or the older 1.11.40 is cuasing a bootloop, the following must be done:

Check device vendor id and product id:
```
lsusb
```
Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 001 Device 016: ID 1949:03c8 Lab126, Inc. KFMUWI
Bus 001 Device 004: ID 1d50:606f OpenMoko, Inc. Geschwister Schneider CAN adapter
Bus 001 Device 003: ID 046d:0825 Logitech, Inc. Webcam C270
Bus 001 Device 002: ID 2109:3431 VIA Labs, Inc. Hub
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub

Here my android device is the Amazon Fire HD tablet. So my vid=1949 and pid=03c8.

Now create a udev rule:
```
sudo nano /etc/udev/rules.d/51-android.rules
```

insert this line:
```
SUBSYSTEM=="usb", ATTR{idVendor}=="1949", ATTR{idProduct}=="03c8", MODE="0666", GROUP="plugdev"
```
Save the file with Ctrl+X
Confirm with Y

Now the device is good to be detected once udev rule is reloaded. 
So, let's do it:
```
sudo udevadm control --reload-rules
```
After this, again check if your device is detected by adb:
```
adb devices
```
List of devices attached
GCC0X80814350GH5  device

So, you are done.

If it still doesn't work, unplug/replug the device.
If it still doesn't work, restart your OS.

Hope that helps you guys out!
