---
title: Rooting Huawei Y5 Prime 2018 (DRA-LX2)
layout: posts.liquid
is_draft: false
published_date: 2024-09-14 20:30:00 +0330
description: Steps to rooting an old Android phone
categories: ['Android', 'Magisk']
---
<p class="notice" style="color: red">
Attention: You are responsible for any damages to your device, read the article with CARE and replicate it after FULLY reading it. 
</p>

## Needed software and hardware
- Your DRA-LX3 phone
- USB Cable with data as well
- A Windows 10 or 11 machine (Have not tested this on Linux)
- Internet connection
  
## Step 1: Enabling Developer mode and installing ADB
First you need to enable developer mode in your device, this is easily done by navigating to `Settings -> System -> About Phone`.  
In this screen click on `Build number` many times until you see `You are now a developer!`.   
Then navigate to `Settings -> System -> Developer options`, enable `OEM Unlocking` and `USB debugging`.  
   
Second you need to install `ADB` on your Windows machine, you can use `winget` to do so using this: 
```bash
winget install --id Google.PlatformTools
```
  
Connect your device via the USB Cable, and run this command, you should see a popup on your phone telling you to give permission. 
```bash
adb devices
```

## Step 2: Bypassing boot-rom protection
### Install (once)
Devices with MTK SoC's have boot-rom protection preventing us from using tools like SP Flash, luckily there are exploits to bypass this, thanks to [MTK Bypass](https://github.com/MTK-bypass). 
First we need to install Python, we can again use `winget` for this task as well by running this command: 
```bash
winget install python.python.3.9
```
Any modern version should work.  
Then you need to install [UsbDk](https://github.com/daynix/UsbDk/releases), click on the link and download the latest MSI file for your machine.  
Finally download [Bypass Utility](https://github.com/MTK-bypass/bypass_utility/archive/refs/heads/master.zip) and [Exploits Collection](https://github.com/MTK-bypass/exploits_collection/archive/refs/heads/master.zip).  
Extract these folders and place the folder containing the `payloads` folder and `default_config.json5` file next to the `src` folder.  
### Usage
Now we can use the tool to unlock the boot-rom protection, notice that you will need to do this a few more times as well. Only repeat these steps. 
1. Turn off your device while it's NOT connected to your machine. 
2. Hold down the <kbd>Volume down</kbd> button on your phone. Do these steps while hold that.
3. Navigate to the folder containing the `main.py` file and run this command: 
```bash
python main.py
```
4. Connect your phone to the machine (still holding the <kbd>Volume down</kbd> button).
You will see an output similar to this: 
```
[2024-09-14 17:04:00.884640] Waiting for device
[2024-09-14 17:04:02.972299] Found device = 0e8d:0003

[2024-09-14 17:04:03.219234] Device hw code: 0x699
[2024-09-14 17:04:03.220232] Device hw sub code: 0x8a00
[2024-09-14 17:04:03.222458] Device hw version: 0xca00
[2024-09-14 17:04:03.223495] Device sw version: 0x0
[2024-09-14 17:04:03.224173] Device secure boot: True
[2024-09-14 17:04:03.225168] Device serial link authorization: True
[2024-09-14 17:04:03.226165] Device download agent authorization: False

[2024-09-14 17:04:03.227163] Disabling watchdog timer
[2024-09-14 17:04:03.231333] Disabling protection
[2024-09-14 17:04:03.267116] Protection disabled
```
Once you see **Protection disabled**, the protection is off for the next boot. You can continue the steps.
  
## Step 3: Downloading the boot image. 
We need to download the device's original boot image to our machine, to do so we need [SP Flash Tool](https://spflashtool.com/), download the software from the link (the latest version), and run `flash_tool.exe`.  
You need the scatter file for your phone to download the boot image, download [this text file](https://gist.githubusercontent.com/alirezaahani/51b94a6d6d570d02bdc6d8c433e3245e/raw/83ba75eacb2356bfe4efa87ecb060fa7404c2d64/MT6739_Android_scatter.txt) and save it somewhere you can relocate.  
Under the `Download` tab in SP Flash tool, locate the option `Scatter-loading File` and choose the address for the text file you downloaded.  
Click the `Download` button there and enter these values for the download options: 
```
Start: 0x15800000
Length: 0x1800000
```
Save the boot image as `boot.img` and save it somewhere safe.

## Step 4: Patch the boot image.
Move the `boot.img` to a SD Card and put in your phone. Turn your phone on.  
Download and install the [Magisk](https://github.com/topjohnwu/Magisk) app from their github page.  
Open the app, in the home page click on `Magisk -> Install`.  
In the new page turn on the `Preserve AVB 2.0/dm-verity` option.  
Select the `Select and patch a file` method and select your `boot.img` file. Magisk will patch the boot image and save it inside the phone. Copy it to your SD Card and move the patched file to your machine. 

## Step 5: Write the boot image.
First we need to get rid of boot-rom protection again. Use **Step 3** again to help.  
After that open SP Flash tool and press <kbd>Ctrl</kbd> + <kbd>Alt</kbd> + <kbd>V</kbd>. From the `Window` menu select `Write Memory`.  
In the `Write Memory` tab, locate `File path:` and select the patched image file from the pervious step.  
Fill the `Begin Address` input with this value: `0x15800000`  
Click `Write Memory` wait, after it's finished, turn on your phone.  
## Step 6: Your device is now rooted!
From now on you will see this message on boot-up: 
```
Your device has failed verification and may not work properly. 
To learn more, visit: 
...
```
Do not worry, just press the power button and your phone will boot-up. Unfortunately I haven't found a way to fix this yet and you will need to do this every time.  
Have fun messing around with your rooted Android phone.  
If you had any problems following the steps contact me and I will try to help if I can.  
I you find any custom roms or ASOP or any other interesting thing to do with this phone, be sure to contact me.  
