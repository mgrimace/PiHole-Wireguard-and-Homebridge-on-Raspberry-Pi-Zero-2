# PiHole-Wireguard-and-Homebridge-on-Raspberry-Pi-Zero-2
These are my install notes for setting up [Cloudblock](https://github.com/chadgeary/cloudblock) and [Homebridge](https://homebridge.io) docker containers on my new Raspberry Pi Zero 2w. Please feel free to contribute notes, suggestions, clarifications, etc. 

**Cloudblock** combines PiHole (adblock) for local ad-blocking (i.e., whole-home adblocking), wireguard for remote ad-blocking (i.e., out-of-home ad-blocking; using split-tunnel DNS over VPN) and cloudflared (DNS over HTTPS) all in docker containers.

**Homebridge** allows my home to recognize my assorted smart devices as HomeKit compatible. With this setup I can access both PiHole and Homebridge on my local network or out-of-home. 

# Table of contents

1. [Equipment](#Equipment)
2. [Build Pi Zero 2](#Build-Pi-Zero-2)
3. [Setup the Pi](#Setup-the-Pi)
4. [Install Cloudblock](#Install-Cloudblock)
5. [Post-install PiHole setup](#PiHole-setup)
6. [Install HomeBridge](#Install-HomeBridge)
7. [Post-install HomeBridge setup](#HomeBridge-setup)
8. [Updating](#Updating)
9. [To Do](#To-Do)
10. [Support this project](#Support-this-project)

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

# Setup the Pi

## Connect to the Pi

- Insert the SD card, snap on the lid, and plug your Pi in to the A/C adapter.
- Your Pi will automatically connect to your wifi. For me, the easiest way to find the Raspberry Pi's IP address was to look for it on my network using my router's app
- I used my wireless router settings to reserve its IP address (i.e., so it doesn't change the IP on me)
- Open Terminal and SSH in using `ssh pi@rasberrypi.local` if you changed your Pi username, replace `pi` with `username`. If you selected allow public-key authentication only, you shouldn't have to use a password to connect.

## Increase the Pi's memory

Prior to running ansible as part of installing Cloudblock, we need to increase the available memory or we run into errors during the install process due to limited free memory on the Pi Zero 2. There are two ways to do this, I've tested both and I'm using ZRam currently:

### Option 1 - enable ZRam

- [Zram](https://www.kernel.org/doc/Documentation/blockdev/zram.txt**) creates compressed RAM based block storage. This compression allows additional memory inside RAM in exchange for the processing power used for compression. This has the benefit of being faster than using the SD card for swap memory.
- To enable, use `sudo apt install zram-tools` 

### Option 2 - increase the swap memory

- Increasing the swap memory designates a portion of the SD card to act as memory. Using the SD card as memory is slow but reliable. To increase the swap memory:

- ```sudo dd if=/dev/zero of=/opt/swap.file bs=1024 count=1048576
  sudo dd if=/dev/zero of=/opt/swap.file bs=1024 count=1048576
  sudo mkswap /opt/swap.file
  sudo chmod 600 /opt/swap.file
  sudo swapon /opt/swap.file
  ```

## (Optional) Add another device that can log-in to the Pi

1. On your other computer that you want to also be able to log into the Pi from, open terminal and use `ssh-keygen -t rsa` to create a new SSH key pair. Alternatively, use one you already have.
2. Copy the contents of the public key using `cat ~/id_rsa.pub`
3. Login to the Pi using your working device, then use `echo [pate public key content here] >> ~/.ssh/authorized.keys`.  If that doesn't work, try `authorized_keys`
4. Alternatively, use a text-editor like `nano ~/.ssh/authorized.keys` and paste your public key content to a new line in that file. If you're working with nano, press `esc` then `$` to wrap long strings of text (like these keys) to make it easy to read. Then hit `control + x` then `y` then `enter` to save your changes.
5. Log out using your old device
6. Log in using your new device, assuming you're using the defaults, you should be able to log-in using `ssh pi@raspberrypi.local` or `ssh -i ~/.ssh/[key name] [username]@[raspberry pi ip]` if you're not using defaults keys and/or user names
7. Repeat steps 1-6 on any other devices you want to add

## (Optional) Create a host file to make it easier to log in if you're not using default keys

- If you're not using the default key name of `id_rsa` (e.g., you created your own name), make a file called `config` in your ~/.ssh/ directory and include the following

- ```
  Host [Raspberry Pi IP]
  IndentityFile ~/.ssh/[key name]
  ```

- This will allow you to connect to your Pi in Terminal by just using `ssh [username]@[raspberry pi ip]`. If you skip this step, you'll need to specify the name of your key file every time you connect, as in: `ssh -i ~/.ssh/[key name] [username]@raspberry pi ip]`

# Install Cloudblock

* Follow the local deployment, raspbian guide here: https://github.com/chadgeary/cloudblock/tree/master/playbooks#raspbian-deployment, for support join their [Discord](https://discord.gg/zmu6GVnPnj), and check out the [video tutorial](https://youtu.be/9oeQZvltWDc)
  * At the #Set Variables step, add your DDNS url if you have one (I got this from my router settings, but not all routers have this functionality)
  * Note, if you did not manually specify a password for the PiHole admin page, you'll need to use `sudo cat /opt/pihole/ph_password` afterwards running ansible to see the password you generated


- After ansible completes, take note of the final output which includes your local and remote PiHole IP addresses, and Wireguard config files. The PiHole IPs will allow you to connect to your PiHole admin portal at home and out-of-home. I made a separate bookmark for each (e.g. PiHole - Home, PiHole - Remote). 
- Use the Wireguard QR codes to setup your mobile devices. I set the profiles to *on-demand* except when connected to my home wifi SSID. That means that as soon as I leave home, Wireguard will connect remotely to continue ad-blocking.
- To download the Wireguard config files to your computer, use the following secure-copy commands. Make sure you are *not* connected by SSH when running this: `scp -r pi@raspberrypi.local:/opt/wireguard/peer*/ [destination on home computer]` For example, I saved them to a folder called RPi in My Documents using: `scp -r pi@raspberrypi.local:/opt/wireguard/peer*/ ~/Documents/pihole_configs`

## Troubleshooting Cloudblock setup

* If you are using swap: Every time you re-run ansible (e.g., for updates), you'll need to re-paste the variables, and use `sudo swapon /opt/swap.file` to turn on the swap to avoid errors prior to running `ansible-playbook...`
* If you reboot by pulling the power cord or by losing power, Cloublock and Wireguard to do not seem to startup properly. To fix this, use `sudo reboot` and it should start back up. 

# PiHole setup

- Go to your router settings, note these steps depend entirely on your own router model 

- Forward port 51820 to your Pi's local IP address to enable Wireguard to work properly
- Set your primary DNS in your DHCP server settings to your Pi's local IP. Leave the secondary DNS blank.

## (optional) Setting up adlists and whitelists

- I like the [pihole list tool](https://github.com/jessedp/pihole5-list-tool) for adding adlists and whitelists, you can install it by SSH back to your Pi, then running `sudo pip3 install pihole5-list-tool --upgrade` . Select the Docker version once it launches, then choose blocklists and whitelists options from the list.

## (optional) Setting up apps that use PiHole API token

- You can use PiHole apps (e.g., [Pi-Hole remote](https://apps.apple.com/nl/app/pi-hole-remote/id1515445551?l=en)) by selecting https://, using your [Raspbery Pi IP] and port: 443 along with your PiHole's API token. I set up two PiHoles in the app *PiHole - local* and *PiHole - remote*. To set up remote, I used https://, 172.18.0.5, and port:443

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

# HomeBridge setup

- log-in to the Homebridge admin console by going to `http://[Raspberry Pi IP]:8581`. There you can monitor, Install, and configure various plugins.
- Add `172.18.0.1/32` to your allowed IPS in your client configs to connect to HomeBridge over wireguard (i.e., to connect remotely when out-of-home)

## (Optional) Add the Pihole plugin to HomeBridge 

- You can add a PiHole plugin to HomeBridge to control your PiHole with Siri. Install the plugin and enter your Pi's API Token, then select SSL, use your [Raspberry Pi IP] as the host, and Port 443

# Updating

## Cloudblock

```# Be in the cloudblock/playbooks directory
cd ~/cloudblock/playbooks

# Pull cloudblock code updates
git pull

# Set customized variables
doh_provider=opendns
dns_novpn=1
wireguard_peers=10
vpn_traffic=dns
docker_network=172.18.0.0
docker_gw=172.18.0.1
docker_doh=172.18.0.2
docker_pihole=172.18.0.3
docker_wireguard=172.18.0.4
docker_webproxy=172.18.0.5
wireguard_network=172.19.0.0

# Remove old containers (service is down until Ansible completes)
sudo docker rm -f cloudflared_doh pihole web_proxy wireguard

# Rerun ansible-playbook
ansible-playbook cloudblock_raspbian.yml --extra-vars="doh_provider=$doh_provider dns_novpn=$dns_novpn wireguard_peers=$wireguard_peers vpn_traffic=$vpn_traffic docker_network=$docker_network docker_gw=$docker_gw docker_doh=$docker_doh docker_pihole=$docker_pihole docker_wireguard=$docker_wireguard docker_webproxy=$docker_webproxy wireguard_network=$wireguard_network"
```

## Homebridge

```
# run these commands from the same directory you created the docker-compose.yml file in
docker-compose pull
docker-compose up -d
```

# Useful commands

- `sudo  reboot` #to reboot pi
- `/usr/bin/vcgencmd measure_temp` #to quickly check temp. I find `sudo reboot` and running scripts can stall/freeze if the temp is too high (>55 C).
- `ssh-keygen -R raspberrypi.local` #If re-create your SD card using your previous/existing keys, you’ll have to delete your fingerprint using this command and generate a new one. If you do so, run this on your home machine prior to ssh.
- `sudo docker ps` #this will show you what docker containers are running. 
- `sudo docker system prune` #If there were errors/issues during the setup, this will tidy up half-installed, empty, incomplete docker containers. 

# To Do

* Testing 32-bit vs 64-bit memory use 
* Figure out how to pass bluetooth to homebridge docker container to control bluetooth devices on the home network.

# Support this project

If you found this guide helpful, please consider buying me a coffee by clicking the link below. I'll do my best to keep this guide up to date and as user-friendly as possible. Thank you and take good care!

[![Donate](https://img.shields.io/badge/Donate-PayPal-green.svg)](https://www.paypal.com/cgi-bin/webscr?cmd=_donations&business=R4QX73RWYB3ZA)
