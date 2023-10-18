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
      image: oznu/homebridge:latest
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

* If `sudo docker-compose up -d` results in an error try running `sudo docker-compose pull` first


# HomeBridge setup

- log-in to the Homebridge admin console by going to `http://[Raspberry Pi IP]:8581`. There you can monitor, install, and configure various plugins.
- Add `172.18.0.1/32` to your allowed IPS in your client configs to connect to HomeBridge over wireguard (i.e., to connect remotely when out-of-home)

## (Optional) Add the Pihole plugin to HomeBridge 

- You can add a PiHole plugin to HomeBridge to control your PiHole with Siri. Install the plugin and enter your Pi's API Token, then select SSL, use your [Raspberry Pi IP] as the host, and Port 443

# Updating the app

```bash
# run these commands from the same directory you created the docker-compose.yml file in
docker-compose pull
docker-compose up -d
```

# 