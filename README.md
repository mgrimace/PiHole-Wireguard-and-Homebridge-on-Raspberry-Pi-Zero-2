# PiHole-Wireguard-and-Homebridge-on-Raspberry-Pi-Zero-2
These are my install notes for setting up [Cloudblock](https://github.com/chadgeary/cloudblock) and [Homebridge](https://homebridge.io) on my new Raspberry Pi Zero 2w.

**Cloudblock** combines PiHole (adblock) for local ad-blocking (i.e., whole-home adblocking), wireguard for remote ad-blocking (i.e., out-of-home ad-blocking; using split-tunnel DNS over VPN) and cloudflared (DNS over HTTPS) all in docker containers.

**Homebridge** allows my home to recognize my assorted smart devices as HomeKit compatible. With this setup I can access both PiHole and Homebridge on my local network or out-of-home. 

# Table of contents

1. [Equipment](#Equipment)
2. [Build Pi Zero 2](#Build-Pi-Zero-2)
3. [Install Cloudblock](#Install-Cloudblock)
4. [Post-install PiHole setup](#Post-install-PiHole-setup)
5. [Install HomeBridge](#Install-HomeBridge)
6. Post-install HomeBridge setup (in progress)

# Equipment

I purchased a Raspberry Pi Zero 2 Starter kit, which included:

- Raspberry Pi Zero 2 W
- A UniPiCase Zero Standard Case
- Mini HDMI to HDMI Adapter
- USB OTG Cable
- 32GB Class 10 SD Card - blank
- 5.1V 2.5A Power Supply
- 2x20 Male Header Strip A small extra if you decide to tinker with the GPIO outside of the case. Solder this on and you can plug in Pi HATs, GPIO cables, etc, just as you would on a normal Pi!
- Set of two heatsinks

For this project, I won't be using the adapters or the header strip.

# Build Pi Zero 2

* Select the largest of the two heatsinks, peel off the sticker backing on the bottom, and attach it to the chip. You don't need the smaller heatsink.
* Open the UniPiCase by pinching the tabs on the bottom and lightly pulling up on the top of the case
* Put the front plate from the UniPiCase onto the Pi Zero 2, then gently push the front plate and Pi Zero 2 down into the bottom of the case. There's a little tab on the left that will hold it in place.
* Leave the top off for now, but you can stick the little rubber feet onto the bottom of the case at this point

## Get the SD card ready

* Go to [raspberrypi.com](raspberrypi.com) and download and install the Raspberry Pi Imager. I'm using a Mac for this install
* I selected Raspberry Pi OS **Lite** **64-bit** by going to "other" section. Before continuing, select advanced options
  * **Enable SSH**: I selected *allow public-key authenitcation* only, and left the default/prefilled option for set authorized_keys for 'pi'. This automatically creates the required authorization keys. That means I don't have to use a password when connecting to my Pi from this computer
  * Set username and password
  * **Set-up wifi:** Set wifi SSID and password
* Write the SD card

## Connect to Pi

* Insert the SD card, snap on the lid, and plug your Pi in to the A/C adapter.
* Your Pi will automatically connect to your wifi. For me, the easiest way to find the Raspberry Pi's IP address was to look for it on my network using my router's app
* I used my wireless router settings to reserve its IP address (i.e., so it doesn't change the IP on me)
* Open Terminal and SSH in using `ssh pi@rasberrypi.local` if you changed your Pi username, replace `pi` with `username`. If you selected allow public-key authentication only, you shouldn't have to use a password to connect.

# Install Cloudblock

* Follow the local deployment, raspbian guide here: https://github.com/chadgeary/cloudblock/tree/master/playbooks#raspbian-deployment, for support join their [Discord](https://discord.gg/zmu6GVnPnj), and check out the [video tutorial](https://youtu.be/9oeQZvltWDc)
  * At the #Set Variables step, add your DDNS url if you have one (I got this from my router settings, but not all routers have this functionality)
  * Note, if you did not manually specify a password for the PiHole admin page, you'll need to use `sudo cat /opt/pihole/ph_password` afterwards running ansible to see the password you generated

* Prior to running the ansible step there's a few things we need to do specific to the Pi Zero 2 hardware:
  * You'll need to increase your swap memory or you'll run into errors during the install process due to limited free memory. After this, you can run the ansible playbook. To increase the swap memory:

- ```sudo dd if=/dev/zero of=/opt/swap.file bs=1024 count=1048576
  sudo dd if=/dev/zero of=/opt/swap.file bs=1024 count=1048576
  sudo mkswap /opt/swap.file
  sudo chmod 600 /opt/swap.file
  sudo swapon /opt/swap.file
  ```


- After ansible completes, take note of the final output which includes your local and remote PiHole IP addresses, and Wireguard config files. The PiHole IPs will allow you to connect to your PiHole admin portal at home and out-of-home. I made a separate bookmark for each (e.g. PiHole - Home, PiHole - Remote). 
- Use the Wireguard QR codes to setup your mobile devices. I set the profiles to *on-demand* except when connected to my home wifi SSID. That means that as soon as I leave home, Wireguard will connect remotely to continue ad-blocking.
- To download the Wireguard config files to your computer, use the following secure-copy commands. Make sure you are *not* connected by SSH when running this: `scp -r pi@raspberrypi.local:/opt/wireguard/peer*/ [destination on home computer]` For example, I saved them to a folder called RPi in My Documents using: `scp -r pi@raspberrypi.local:/opt/wireguard/peer*/ ~/Documents/RPi`

## Troubleshooting Cloudblock setup

* Every time you re-run ansible (e.g., for updates), you'll need to re-paste the variables, and use `sudo swapon /opt/swap.file` to turn on the swap to avoid errors prior to running `ansible-playbook...`

# Post-install PiHole setup

Go to your router settings, note these steps depend entirely on your router model

- Forward port 51820 to your Pi's local IP address to enable Wireguard to work properly
- Set your primary DNS in your DHCP server settings to your Pi's local IP. Leave the secondary DNS blank.
- I like the [pihole list tool](https://github.com/jessedp/pihole5-list-tool) for adding adlists and whitelists, you can install it by SSH back to your Pi, then running `sudo pip3 install pihole5-list-tool --upgrade` . Select the Docker version once it launches, then choose blocklists and whitelists options from the list.

# Install HomeBridge

* Install Docker Compose `sudo apt-get -y install docker-compose`

* Make a new folder for our docker compose file `mkdir /home/pi/homebridge`, and navigate to that folder `cd /home/pi/homebridge`

* Create a Docker Compose file `nano docker-compose.yml`

* Copy and paste the official Docker Compose Manifest from Homebridge found [here](https://github.com/homebridge/homebridge/wiki/Install-Homebridge-on-Docker#step-2-create-docker-compose-manifest), or use:

* ```
  version: '2'
  services:
    homebridge:
      image: oznu/homebridge:ubuntu
      container_name: homebridge
      restart: always
      network_mode: host
      environment:
        - HOMEBRIDGE_CONFIG_UI_PORT=8581
      volumes:
        - ./homebridge:/homebridge
  ```

* After pasting, press `control+x` to exit, `y` to save, then `enter` to confirm without changing the file name

* Then run `sudo docker-compose up -d` to install Homebridge from the Docker Compose file.



# Post-install HomeBridge setup
