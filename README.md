# PiHole-Wireguard-and-Homebridge-on-Raspberry-Pi-Zero-2
These are my install notes for setting up [Cloudblock](https://github.com/chadgeary/cloudblock) and [Homebridge](https://homebridge.io) docker containers on my new Raspberry Pi Zero 2w. Please feel free to contribute notes, suggestions, clarifications, etc. 

**Cloudblock** combines PiHole (adblock) for local ad-blocking (i.e., whole-home adblocking), wireguard for remote ad-blocking (i.e., out-of-home ad-blocking; using split-tunnel DNS over VPN) and cloudflared (DNS over HTTPS) all in docker containers.

**Homebridge** allows my home to recognize my assorted smart devices as HomeKit compatible. With this setup I can access both PiHole and Homebridge on my local network or out-of-home. 

# Table of contents

1. [Equipment](#Equipment)
2. [Build Pi Zero 2](#Build-Pi-Zero-2)
3. [Setup the Pi](#Setup-the-Pi)
4. [Install Cloudblock](#Install-Cloudblock)
5. [Router setup](#Router-setup)
6. [Useful Pihole addons](#Useful-Pihole-Addons)
7. [Install HomeBridge](#Install-HomeBridge)
8. [Post-install HomeBridge setup](#HomeBridge-setup)
9. [Updating Pihole and Homebridge](#Updating-the-apps)
10. [Useful commands](#Useful-commands)
11. [Support this project](#Support-this-project)

# Equipment

I purchased a Raspberry Pi Zero 2 Starter kit, which included:

- Raspberry Pi Zero 2 W
- A UniPiCase Zero Standard Case
- Mini HDMI to HDMI Adapter
- USB OTG Cable
- 32GB Class 10 SD Card - blank
- 5.1V 2.5A Power Supply
- 2x20 Male Header Strip 
- Set of two heatsinks

For this project, I won't be using the adapters or the header strip.

# Build Pi Zero 2

* Select the largest of the two heatsinks, peel off the sticker backing on the bottom, and attach it to the chip. You don't need the smaller heatsink.
* Open the UniPiCase by pinching the tabs on the bottom and lightly pulling up on the top of the case
* Put the front plate from the UniPiCase onto the Pi Zero 2, then gently push the front plate and Pi Zero 2 down into the bottom of the case. There's a little tab on the left that will hold it in place.
* Leave the top off for now, but you can stick the little rubber feet onto the bottom of the case at this point

## Get the SD card ready

* Go to [raspberrypi.com](raspberrypi.com) and download and install the Raspberry Pi Imager. I'm using a Mac for this install
* I selected Raspberry Pi OS **Lite** **32-bit** by going to "other" section. Note, I tested both 32-bit and 64-bit and don't see much difference in speed or memory use, feel free to use Lite 64-bit if you prefer. 
  * **(Optional) additional reading:** There's an exceptionally detailed techincal writeup on the Pi Zero 2 [here](https://github.com/ThomasKaiser/Knowledge/blob/master/articles/Quick_Review_of_RPi_Zero_2_W.md) that suggests 32-bit systems are more memory efficient on a memory-limited system such as this.

* Before continuing, select advanced options:
  * **Enable SSH**: I selected *allow public-key authenitcation* only, and left the default/prefilled option for set authorized_keys for 'pi'. This automatically creates the required authorization keys. That means I don't have to use a password when connecting to my Pi from this computer
  * Set username and password
  * **Set-up wifi:** Set wifi SSID and password
* Write the SD card

# Setup the Pi

## Connect to the Pi

- Insert the SD card into the Pi, snap on the lid, and plug your Pi in to the A/C adapter using the right-most slot
- Your Pi will automatically connect to your wifi. For me, the easiest way to find the Raspberry Pi's IP address was to look for it on my network using my router's app
- **(Recommended) Set a static IP for the Pi** I used my wireless router settings to reserve its IP address (i.e., so the Pi doesn't change its IP on me)
- Open Terminal and connect to it remotely by using SSH. To do so type or copy/paste `ssh pi@rasberrypi.local`. If you changed your Pi username, replace `pi` with `username`. If you selected allow public-key authentication only, you shouldn't have to use a password to connect.

## Increase the Pi's memory

Prior to running ansible as part of installing Cloudblock, we need to increase the available memory or we run into errors during the install process due to limited free memory on the Pi Zero 2. There are two ways to do this, I've tested both and I'm using ZRam currently:

### Option 1 - enable ZRam

- [Zram](https://www.kernel.org/doc/Documentation/blockdev/zram.txt**) creates compressed RAM based block storage. This compression allows additional memory inside RAM in exchange for the processing power used for compression. This has the benefit of being faster than using the SD card for swap memory.
- To enable, use `sudo apt install zram-tools` 
- **(optional)** increase Pi's tendency to use swap now that we are using Zram (for more information, see [source](https://haydenjames.io/raspberry-pi-performance-add-zram-kernel-parameters/) for more details). 

  * Use `sudo nano /etc/sysctl.conf` to add the following to the end of your **/etc/sysctl.conf** file:

    ```bash
    vm.vfs_cache_pressure=500
    vm.swappiness=100
    vm.dirty_background_ratio=1
    vm.dirty_ratio=50
    ```

    Then enable with:

    ```bash
    sudo sysctl --system
    ```

### Option 2 - increase the swap memory

- Increasing the swap memory designates a portion of the SD card to act as memory. Using the SD card as memory is slow but reliable. To increase the swap memory:

- ```bash
  sudo dd if=/dev/zero of=/opt/swap.file bs=1024 count=1048576
  sudo mkswap /opt/swap.file
  sudo chmod 600 /opt/swap.file
  sudo swapon /opt/swap.file
  ```

## (Optional) Add another device that can log-in to the Pi

1. On your other computer that you want to also be able to log into the Pi from, open terminal and use `ssh-keygen -t rsa` to create a new SSH key pair. Alternatively, use one you already have.
2. Copy the contents of the public key using `cat ~/.ssh/id_rsa.pub`
3. Login to the Pi using your working device, then use `echo [paste public key content here] >> ~/.ssh/authorized_keys`. 
4. Alternatively, use a text-editor like `nano ~/.ssh/authorized_keys` and paste your public key content to a new line in that file. If you're working with nano, press `esc` then `$` to wrap long strings of text (like these keys) to make it easy to read. Then hit `control + x` then `y` then `enter` to save your changes.
5. Log out using your old device
6. Log in using your new device, assuming you're using the defaults, you should be able to log-in using `ssh pi@raspberrypi.local` or `ssh -i ~/.ssh/[key name] [username]@[raspberry pi ip]` if you're not using defaults keys and/or user names
7. Repeat steps 1-6 on any other devices you want to add

## (Optional) Create a host file to make it easier to log in if you're not using default keys

- If you're not using the default key name of `id_rsa` (e.g., you created your own name), make a file called `config` in your home computer's ~/.ssh/ directory and include the following

- ```bash
  Host [Raspberry Pi IP]
  IndentityFile ~/.ssh/[key name]
  ```

- This will allow you to connect to your Pi in Terminal by just using `ssh [username]@[raspberry pi ip]`. If you skip this step, you'll need to specify the name of your key file every time you connect, as in: `ssh -i ~/.ssh/[key name] [username]@raspberry pi ip]`

# Install Cloudblock

* Follow the local deployment, raspbian guide here: https://github.com/chadgeary/cloudblock/tree/master/playbooks#raspbian-deployment, for support join their [Discord](https://discord.gg/zmu6GVnPnj), and check out the [video tutorial](https://youtu.be/9oeQZvltWDc):
  
* ```bash
  # Ansible + Git
  sudo apt update && sudo apt -y upgrade
  sudo apt install git python3-pip
  pip3 install --user --upgrade ansible
  
  # Add .local/bin to $PATH
  echo PATH="\$PATH:~/.local/bin" >> .bashrc
  source ~/.bashrc
  
  # Install the community ansible collection
  ansible-galaxy collection install community.general
  
  # Optionally, reboot the raspberry pi
  # This may be required if the system was months out date before installing updates!
  sudo reboot
  
  # Clone the cloudblock project and change to playbooks directory
  git clone https://github.com/chadgeary/cloudblock && cd cloudblock/playbooks/
  
  # Set Variables
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
  
  # Optional (e.g. your DDNS hostname)
  wireguard_hostname=example.com
  
  # Want to set your own pihole password instead of something randomly generated?
  sudo mkdir -p /opt/pihole
  echo "somepassword" | sudo tee /opt/pihole/ph_password
  sudo chmod 600 /opt/pihole/ph_password
  
  # Execute playbook via ansible
  ansible-playbook cloudblock_raspbian.yml --extra-vars="doh_provider=$doh_provider dns_novpn=$dns_novpn wireguard_peers=$wireguard_peers vpn_traffic=$vpn_traffic docker_network=$docker_network docker_gw=$docker_gw docker_doh=$docker_doh docker_pihole=$docker_pihole docker_wireguard=$docker_wireguard docker_webproxy=$docker_webproxy wireguard_network=$wireguard_network wireguard_hostname=$wireguard_hostname"
  
  # See Playbook Summary output for Pihole WebUI URL and Wireguard Client files
  ```
  
  * **Optional**: At the `# Set Variables` step, add your DDNS url if you have one (I got this from my router settings, but not all routers have this functionality). You can do this by adding the line `wireguard_hostname=[your personal DDNS address]`
  * Note, if you did not manually specify a password for the PiHole admin page, you'll need to use `sudo cat /opt/pihole/ph_password` afterwards running ansible to see the password you generated

- After ansible completes, **take note of the final output which includes your local and remote PiHole IP addresses, and Wireguard config files**. The PiHole IPs will allow you to connect to your PiHole admin portal at home and out-of-home. I made a separate bookmark for each (e.g. PiHole - Home, PiHole - Remote). 
- Use the Wireguard QR codes to setup your mobile devices. I set the profiles to *on-demand* except when connected to my home wifi SSID. That means that as soon as I leave home, Wireguard will connect remotely to continue ad-blocking.
- To download the Wireguard config files to your computer, use the following secure-copy commands. Make sure you are *not* connected by SSH when running this on your home computer: `scp -r pi@raspberrypi.local:/opt/wireguard/peer*/ [destination on home computer]`
- For example, I saved them to a folder called pihole_configs in My Documents using: `scp -r 'pi@raspberrypi.local:/opt/wireguard/peer*' ~/Documents/pihole_configs`

### Important additional step

Since the Raspberry Pi's DHCP server (my router) points to the PiHole container, I need to ensure that the Raspberry Pi's host DNS is not set via DHCP (i.e., set to itself, thus creating a non-working loop). This happened to me after rebooting the Pi, after which all the containers were working but my devices did not have internet. To do so, follow these steps: https://github.com/chadgeary/cloudblock/tree/master/playbooks#faqs	

```bash
# If the Raspberry Pi's DHCP server points to the Pihole container, ensure the Raspberry Pi's host DNS is not set via DHCP, e.g.:
# backup DHCP client conf
sudo cp /etc/dhcpcd.conf /etc/dhcpcd.conf.$(date +%F_%T)

# Disable DNS via DHCP
sudo sed -i 's/option domain_name_servers, domain_name, domain_search, host_name/option domain_name, domain_search, host_name/' /etc/dhcpcd.conf

# Set a hardcoded DNS server IP - replace 1.1.1.1 with your choice of DNS.
sudo sed -i '0,/#static domain_name_servers=.*/s//static domain_name_servers=1.1.1.1/' /etc/dhcpcd.conf

# if Raspberry Pi is wireless, disconnect/reconnect link
sudo bash -c 'ip link set wlan0 down && ip link set wlan0 up' &
```



## Troubleshooting Cloudblock setup

* If you are using regular swap: every time you re-run ansible (e.g., for updates), you'll need to re-paste the variables, and use `sudo swapon /opt/swap.file` to turn on the swap to avoid errors prior to running `ansible-playbook...`
* If you reboot by pulling the power cord or by losing power, Cloublock and Wireguard to do not seem to startup properly. To fix this, use `sudo reboot` and it should start back up. 
* After updating the Raspberry Pi and/or rebooting, I ran into an issue where everything was working (e.g., PiHole, etc.) but my devices were not able to access the internet. This is an issue with cloudflared_doh (see this [issues](https://github.com/cloudflare/cloudflared/issues/23#issuecomment-1161211222) thread), and restarting it manually fixed the problem. To do so, use: `sudo docker restart cloudflared_doh`. However, I have now added an additional step above to hopefully prevent this from happening.

# Router setup

- Go to your router settings, note these steps depend entirely on your own router model 
- Forward port 51820 to your Pi's local IP address to enable Wireguard to work properly
- Set your primary DNS in your DHCP server settings to your Pi's local IP. **Leave the secondary DNS blank.** 

# Useful Pihole Addons

## Easily add adlists and whitelists

I like the [pihole list tool](https://github.com/jessedp/pihole5-list-tool) for adding adlists and whitelists, you can install it by SSH back to your Pi, then running `sudo pip3 install pihole5-list-tool --upgrade` . Select the Docker version once it launches, then choose the blocklists and whitelists options from the list that appeal to you.

## Set up apps that use PiHole API token

You can use PiHole apps (e.g., [Pi-Hole remote](https://apps.apple.com/nl/app/pi-hole-remote/id1515445551?l=en)) by selecting https://, using your [Raspbery Pi IP] and port: 443 along with your PiHole's API token. I set up two PiHoles in the app *PiHole - local* and *PiHole - remote*. To set up remote, I used https://, 172.18.0.5, and port:443

## Use the list tool to check which lists you're actually using

The [Pihole Adlist Tool](https://github.com/yubiuser/pihole_adlist_tool) is a script that will analyze 30 days worth of adblocking to see which lists you're actually using. After running the script, it can automatically disable any lists that aren't being used. To run it, ssh into your PiHole then run:

```bash
 wget https://github.com/yubiuser/pihole_adlist_tool/archive/refs/tags/2.6.3.zip
 unzip 2.6.3.zip
 rm -r 2.6.3.zip
 cd pihole_adlist_tool-2.6.3
 sudo ./pihole_adlist_tool
```

Then follow the prompts, noting that it may take some time to run. **Note** that you might want to check for the latest release of the tool, as v2.6.3 might be out of date when you read this.

If PiHole is all you wanted, then you can stop here. If you're interested in also adding HomeBridge read on!

# Install HomeBridge

* Install Docker Compose `sudo apt-get -y install docker-compose`

* Reboot now using `sudo reboot`, then reconnect using `ssh pi@raspberrypi.local`. 

* Make a new folder for our docker compose file `mkdir /home/pi/homebridge`, and navigate to that folder `cd /home/pi/homebridge`

* Create a Docker Compose file `nano docker-compose.yml`

* Copy and paste the official Docker Compose Manifest from Homebridge found [here](https://github.com/homebridge/homebridge/wiki/Install-Homebridge-on-Docker#step-2-create-docker-compose-manifest), or use:

* ```dockerfile
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

- log-in to the Homebridge admin console by going to `http://[Raspberry Pi IP]:8581`. There you can monitor, install, and configure various plugins.
- Add `172.18.0.1/32` to your allowed IPS in your client configs to connect to HomeBridge over wireguard (i.e., to connect remotely when out-of-home)

## (Optional) Add the Pihole plugin to HomeBridge 

- You can add a PiHole plugin to HomeBridge to control your PiHole with Siri. Install the plugin and enter your Pi's API Token, then select SSL, use your [Raspberry Pi IP] as the host, and Port 443

# Updating the apps

## Cloudblock

- Be sure to run `sudo apt update && sudo apt -y upgrade` first to update your Raspberry Pi. `sudo reboot`
- Then, remove your Pi-Hole's DNS from your router settings (so you still temporarily still have internet once you remove the old containers). 
- Then run the following:

```bash
# Be in the cloudblock/playbooks directory
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
ansible-playbook cloudblock_raspbian.yml --extra-vars="doh_provider=$doh_provider dns_novpn=$dns_novpn wireguard_peers=$wireguard_peers vpn_traffic=$vpn_traffic docker_network=$docker_network docker_gw=$docker_gw docker_doh=$docker_doh docker_pihole=$docker_pihole docker_wireguard=$docker_wireguard docker_webproxy=$docker_webproxy wireguard_network=$wireguard_network wireguard_hostname=$wireguard_hostname"
```

Once Pi-Hole is successfullly updated, you can re-enter it's IP to your router's DNS settings (I do this in DHCP)

## Homebridge

```bash
# run these commands from the same directory you created the docker-compose.yml file in
docker-compose pull
docker-compose up -d
```

# Useful commands

These are some useful commands I've come across in my learning and testing

- `sudo docker exec pihole pihole updateGravity` update gravity database 
- `sudo  reboot` #to reboot pi
- `/usr/bin/vcgencmd measure_temp` #to quickly check temp. I find `sudo reboot` and running scripts can stall/freeze if the temp is too high (>55 C).
- `ssh-keygen -R raspberrypi.local` #If re-create your SD card using your previous/existing keys, you’ll have to delete your fingerprint using this command and generate a new one. If you do so, run this on your home machine prior to ssh.
- `sudo docker ps` #this will show you what docker containers are running. 
- `sudo docker system prune -a -f` #If there were errors/issues during the setup, this will tidy up half-installed, empty, incomplete docker containers. Useful to run after a few updates.

# Support this project

If you found this guide helpful, please consider buying me a coffee by clicking the link below. I'll do my best to keep this guide up to date and as user-friendly as possible. Thank you and take good care!

[![Donate](https://img.shields.io/badge/Donate-PayPal-green.svg)](https://www.paypal.com/cgi-bin/webscr?cmd=_donations&business=R4QX73RWYB3ZA)
