---
title: "My Hackintosh Setup"
date: 2019-11-23T19:41:00-03:00
summary: "How I set up a Hackintosh using my day-to-day hardware"
draft: false
---

## Important

These are the steps I followed in order to get Mac OS High Sierra 10.13.6 running in my PC. These may not work (and probably won't) on your machine if your build contains different components (like a different motherboard). 

This is not a proper guide or tutorial, just a reference for me or for anyone facing issues with a build like mine. You will need to read a bit about Hackintosh in order to get the most out of these notes.

## My Build

- **CPU**: Ryzen 5 3600 (Stock)
- **Motherboard**: ASUS Crosshair VI Hero - BIOS v7601
- **RAM**: 2x T-Force 3000 MHz (Grey)
- **GPU**: Gigabyte G1 Gaming GTX 1070
- **PSU**: XFX 650w 80+ Bronze Core Edition
- **CPU Cooler**: Cooler Master Hyper 212 Evo
- **Case**: NZXT H440 (Green)
- **Keyboard**: Corsair K95 Platinum RGB
- **Mouse**: Logitech G305 Wireless
- **Mic**: Blue Snowball USB
- **Audio**
  - **Front**: Sennheiser HD 569
  - **Rear**: Admiral Speaker Combo (Generic 3.5mm)
- **Monitor(s)**
  - **Main**: LG 29 UM68-P (via Display Port)
  - **Secondary**: Samsung 43' 4K (via HDMI 2.0)

## How - To

Mainly I followed the [Vanilla OSX Guide](https://vanilla.amd-osx.com/) with some differences:

- **Kexts**: My final Kext configuration is the following (inside `EFI\CLOVER\kexts\Other`):
  - `FakeSMC.kext` (Obligatory)
  - `NullCPUManagement.kext` (Obligatory)
  - `Lilu.kext` (Obligatory)
  - `WhateverGreen.kext` (Obligatory)
  - `AppleALC.kext` (Makes Front and Monitor audio work)
  - `GenericUSBXHCI.kext` (Makes mouse and keyboard work)
  - `USBInjectAll.kext` (Makes mouse and keyboard work)
  - `SmallTreeIntel82576.kext` (Makes LAN work)

All the kexts used are available for download on the [kext repo](https://1drv.ms/f/s!AiP7m5LaOED-m-J8-MLJGnOgAqnjGw) provided and maintained by [Goldfish64](https://github.com/Goldfish64/).

Remember to download the default `config.plist` for Ryzen (17h) from [here](https://github.com/AMD-OSX/AMD_Vanilla) and replace the one in your PenDrive.

- **Clover**: Downgrade your Clover version to [r5092](https://cdn.discordapp.com/attachments/263757191608139779/636426276684562449/Clover_r5092.zip) since it has compatibility issues with `SmallTreeIntel82576`. Just replace the files inside your PenDrive and overwrite if asked to.

## Installing Mac OS

Now you can install Mac OS following the Vanilla guide. Once you finished and you are inside the OS, keep in mind the following:

- **Web Drivers**: Run the following command inside a terminal to install the Nvidia Web Drivers:

  `bash <(curl -s https://vulgo.github.io/webdriver) 387.10.10.10.40.113`

- **Clover Config**:
  - Once the installation finished, I mounted the EFI partition of my main Hard Drive and copied the `EFI` folder inside my PenDrive to it.
  - Add the flag `alcid=1` to the boot flags in the `Boot` menu.
    - *(Optional) Set the boot drive to your main hard drive and remove the timeout (set it to 0 seconds)*.
  - Enable `NvidiaWeb` in the `System Parameters` menu.
  - Disable all flags inside the `Graphics` menu.
  - Install the `EmuVariableUefi` driver from the `Install Drivers` menu.

- **Native USB**: I don't know if this makes a difference but I used anyway. Just run the following command in a terminal:

  `curl -s -o ~/Desktop/ryzenusbfix.sh https://raw.githubusercontent.com/XLNCs/ryzenusbfix/master/ryzenusbfix.sh && chmod +x ~/Desktop/ryzenusbfix.sh && ~/Desktop/ryzenusbfix.sh`

## Notes

Mostly everything works, except for the rear audio (Front Audio and Monitor Speakers work fine) and the GPU performance is pretty bad (try resizing a window). Also, OS updates from the App Store fail to install, so don't bother downloading them.
