# Tesla Cam - An experimental application to repair, store and upload Tesla dash cam footage

## Current capabilities

- [Backup] Storage of Tesla Cam videos
- [Dropbox] Upload videos to Dropbox when a internet connection is available
- [Remote] Basic Mobile App (Web UI) which lets you view videos on your phone, download videos and ability to enable/disable services at will. Available on port 3000 on the IP address of your Pi (Temporarily disabled)
- [Security] Services now run as the pi user and all super user commands are whitelisted
- [Housekeeping] System will delete RecentClips that are X days old (default is two).

## Overview

As of late 2018 Tesla released V9 which among a number of improvements included dash cam functionality. This works by placing a suitably sized USB drive in one of the available USB ports at the front of the vehicle (Model S).

One drawback of this system, not uncommon in dash cams, there's no easy way to push this video to the 'cloud' - nor any capability to view in near real-time. This project aims to make this possible.

Using a couple of tricks I've learned through tinkering with various single board computers, it is possible emulate a USB drive on the fly. In essence we are going emulate a USB drive, and periodically store the data on the SDHC. Once we have the video we can do what ever we'd like - maybe live stream, upload to your favourite cloud provider or simply backup the files when you return home.

## Hardware Requirements

1. 2017 (AP 2.5) or beyond Tesla
2. Raspberry Pi Zero W (only this model is supported)
3. A wireless access point within reasonable distance of the Pi (mobile phone, home router etc)
4. A sufficiently large SDHC card with the fastest write speeds you can find, at least 16Gig, ideally the largest you can buy.
5. High quality short USB A to USB Micro cable - Anker is quite decent
6. Optional, a case to house the Raspberry Pi - anything with ventilation would be fine

## Software Requirements

1. 2018-11-13-raspbian-stretch-lite or later
2. Etcher to write the disk image to the SDHC card (dd, win32diskimager etc etc will also work)
3. Docker
4. OTG Mode enabled in the boot configuration

## Instructions

### On your desktop computer

1. [Download](https://www.raspberrypi.org/downloads/raspbian/) and burn the latest "lite" Raspbian to a suitable SDHC card using [Etcher](https://www.balena.io/etcher/) (or equivalent)
1. Modify the `/boot` partition to [enable USB OTG](https://gist.github.com/gbaman/975e2db164b3ca2b51ae11e45e8fd40a).
   - Add `dtoverlay=dwc2` as a new line to the bottom of `config.txt`
   - Enable g_mass_storage and dw2 by adding `modules-load=dwc2,g_mass_storage` right after `rootwait` in `cmdline.txt`
1. Add your WIFI settings to a new file called `wpa_supplicant.conf` containing the information below. Chane county, ssid and pask accordingly.
   ```
   country=US
   ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
   update_config=1
   
   network={
       ssid="your_real_wifi_ssid"
       scan_ssid=1
       psk="your_real_password"
       key_mgmt=WPA-PSK
   }
   ```
1. Enable ssh by adding an empty file called `ssh` on the `/boot` partition

### On your TeslaCam Pi (via SSH)

1. Plug the Pi Zero W into the Tesla media USB ports (the front ports). Make sure you use the data port on the Pi, google if you are unsure.
1. Connect to the Pi
   ```
   $ ssh pi@raspberrypi.local
   ```
1. Execute the get-teslacam script, this should install everything you need to get up and running.

    ```
    $ GET_TESLACAM=`mktemp` \
    curl -fsSL https://git.io/JeWlq -o ${GET_TESLACAM} && \ 
    sh ${GET_TESLACAM} && \
    rm ${GET_TESLACAM}
    ```

1. Once the automatic configuration completes the car should detect the Pi as a USB drive.

### Optionally install extra services

#### Rsync

1. Generate a ssh key for the rsync service
    ```
    $ docker run \
    --rm \
    -v teslacam_rsync_ssh:/root/.ssh \
    --entrypoint "ssh-keygen" \
    teslacam/dashcam-rsync-upload \
    -f /root/.ssh/id_rsa -q -N ""
    ```

1. Run rsync upload service initial setup to copy making sure to update `user@server` to where you want to upload your key
    ```
    $ docker run \
    --rm \
    -it \
    -v teslacam_rsync_ssh:/root/.ssh \
    --entrypoint "ssh-copy-id" \
    teslacam/dashcam-rsync-upload \
    user@server

1. Run the following command after you update the `RSYNC_TARGET`
    ```
    $ docker run \
    --restart=always \
    -d \
    -v teslacam_rsync_ssh:/root/.ssh \
    -v ${HOME}/teslacam/video:/video \
    -e "RSYNC_TARGET=user@server:~/TeslaCam" \
    --name rsync-upload \
    teslacam/dashcam-rsync-upload
    ```

#### Dropbox Upload

1. Obtain a dropbox token for your account
1. Configure the uploader container by running
    ```
    $ docker run \
    --rm \
    -it \
    -v dropbox_uploader_config:/config \
    --entrypoint=./dropbox_uploader.sh \
    teslacam/dropbox-uploader \
    -f /config/dropbox_uploader.conf
    ```
1. Run the container and set it to restart always
    ```
    docker run \
    --restart=always \
    -v dropbox_uploader_config:/config \
    --name dropbox-uploader \
    teslacam/dropbox-uploader
    ```

# Research & notes

- Tesla V9 Dashcam records up to one hour, in a circular buffer type fashion split into one minute increments
- One hour of footage uses approximately 1.8GiB of storage (over and above any emergency recordings)
- Each one minute increment of video is around 28MiB
- Emergency recordings are 10 minutes at most
- Copying 27 minutes (around 800MiB of data) of footage from a disk image to the ext4 file system takes approximately 4.2 minutes. SDHC class 10
- Time to unmount, repair and copy ~30 minutes of footage is around 4 minutes. In this test the file system wasn't corrupt.
- The Dash cam, USB & 12V ports only operate in the following situations
  - The car is powered on by unlocking the vehicle
  - Climate control is left on when you leave the car
  - It would appear as of V9 the USB ports are powered whilst charging (TBC). May not apply if you use range mode.
  - Sentry mode is enabled
- The Tesla Dash cam tends to be vastly clearer than a interior camera, particularly at night - very easy to make out number plates.
- FAT32, the file system supported by Tesla, cannot be mounted twice without corruption (ie, Real Time streaming is not possible, though near real-time with a 1 minute lag is)
- The car will cut off power to the USB ports without warning, this can cause corruption of video files and any file systems which can not tolerate power loss. This is a tricky issue as there are number of caches (software and hardware) that need to be flushed before power is removed.
- Lipo batteries are not advised within the cabin, temperatures of over 60c have been reported in summer.

## Approach

Primarily there is a trade-off between lost video vs accessibility (our ability to do something useful with the captured footage). To download the Tesla Dash cam video we need to temporarily stop the recording, as Dash cam records in 1 minute increments we are likely to lose at least this much video - possibly more, possibly less depending on timing.

The second concern is we have no signal for when the car will be powered down - ie, you've parked up for the day - the longer we allow the car to record, the higher the possibility that video will be "trapped" in the vehicle till you next power up.

Finally to enable capabilities such as near-real-time monitoring or streaming that video must be transferred to the Pi as quickly as possible. The longer the car records, the longer it takes to transfer - and so on.

To mitigate the issue we need to pick a comfortable number of minutes, say between 10-30 minutes. To add to the fun, we must minimise the duration the car is not recording - to this end we need to switch out our emulated USB drives as quickly as possible which can be done by using two (or more) images swapped over whilst the video files are transferred across.

With all this in mind, logically speaking the following steps need to be followed

- When the Pi powers up
  _ Create or mount two disk images
  _ Scan disk images for errors, and repair
  _ If images contain any videos copy them to the Pi
  _ Unmount both images from the PI
- In a loop pick one disk image
  _ Mount the image allowing the vehicle to begin recording
  _ Wait 30 minutes to accumulate video
  _ Unmount the image from the car
  _ Mount the second Image for the car to record
  _ Scan and fix any errors on the first image
  _ Mount the first image on the Pi
  _ Move all video onto the Pi
  _ Unmount the first image on the Pi

## TODO

- [Streaming] Experiment with streaming, it's trivial to stream to youtube with FFMPEG
- [System] Reverse VPN so the PI is accessible irrespective of the cars location (assuming WIFI is available)
- [Remote] Remote configuration
- [Remote] Infinite pagination
- [Dropbox] Prioritise emergency video upload
- [Github] Explain setup instructions (more detail required)
- [Github] Write decent installation script to automatically configure the application on a Pi
- [System] Use a read only file system to avoid corruption of the operating system
- [System] Make performance metrics more useful (time to upload video etc)
- [System] Improve logging
- [Thoughts] Automatic WiFi hotspot on first boot
- [Me] Buy Tesla Roadster

## Referrals, and coffee

- Tesla Referral link for 1000 free supercharging miles if you buy a new Tesla: https://ts.la/miles16015
- If this was useful and you fancy sending some e-coffee, check out https://paypal.me/milesburton1337
