# Coziee - Home Automation System


## Table of Contents <!-- omit in toc -->
- [1. Install & Configure Raspberry Pi OS](#1-install--configure-raspberry-pi-os)
- [2. Install Docker CE & Docker Compose on Raspberry Pi](#2-install-docker-ce--docker-compose-on-raspberry-pi)
- [3. Flashing CC2531 USB stick at Raspberry Pi](#3-flashing-cc2531-usb-stick-at-raspberry-pi)
- [4. Run `Coziee` containers on Raspberry Pi](#4-run-coziee-containers-on-raspberry-pi)
- [5. Configure `mosquitto` container](#5-configure-mosquitto-container)
- [6. Configure `zigbee2mqtt` container](#6-configure-zigbee2mqtt-container)
- [7. Configure `homebridge` container](#7-configure-homebridge-container)
- [8. Configure `nodered` container](#8-configure-nodered-container)

## 1. Install & Configure Raspberry Pi OS

* [Download Raspberry Pi OS 64-bit lite](https://downloads.raspberrypi.org/raspios_lite_arm64/images/)
* [Install Raspberry Pi OS using Raspberry Pi Imager](https://www.raspberrypi.org/software/)
* [Setting up a Raspberry Pi headless](https://www.raspberrypi.org/documentation/configuration/wireless/headless.md)
* [Enable SSH on a headless Raspberry Pi](https://www.raspberrypi.org/documentation/remote-access/ssh/README.md)
* SSH into Raspberry Pi (Basic Setup)
    ```shell
    $ ssh pi@raspberrypi.local
    # Default Password: raspberry
    $ passwd

    $ sudo raspi-config
    # System Options > Hostname
    # Interface Options > SSH > Yes
    # Localosation Options > Timezone
    # Localosation Options > WLAN Country
    # Advanced Options > Expand Filesystem
    # Update

    $ iwconfig
    $ cat /etc/wpa_supplicant/wpa_supplicant.conf
    $ df -h

    # Update package list
    $ sudo apt update
    # Upgrade all installed packages to latest versions
    $ sudo apt full-upgrade

    # Reboot
    $ sudo shutdown -r now
    ```
* Generate key pairs and copy public key to Raspberry Pi
    ```shell
    $ cd ~/.ssh
    $ ssh-keygen -t ed25519 -f id_ed25519_rpi
    # ssh-keygen -t rsa -b 4096 -f id_rsa_rpi

    $ ssh-add -K ~/.ssh/id_ed25519_rpi
    $ ssh-copy-id -i ~/.ssh/id_ed25519_rpi pi@raspberrypi.local
    $ ssh pi@raspberrypi.local
    ```

## 2. Install Docker CE & Docker Compose on Raspberry Pi

* Install Docker CE on Raspberry Pi OS
    ```shell
    $ curl -fsSL https://get.docker.com -o get-docker.sh
    $ DRY_RUN=1 sh ./get-docker.sh
    $ sudo sh get-docker.sh
    $ rm get-docker.sh

    # Add login user to docker group
    $ sudo usermod -aG docker $(whoami)
    $ grep docker /etc/group

    # Reboot to take effect
    $ sudo shutdown -r now

    # Verify docker is installed
    $ docker version

    # Test with `hello-world` image
    $ sudo docker run --rm hello-world
    ```
* Install Docker Compose on Raspberry Pi OS
    ```shell
    $ sudo apt-get install -y libffi-dev libssl-dev
    $ sudo apt-get install -y python3 python3-pip
    $ sudo apt-get remove python-configparser
    $ sudo pip3 -v install docker-compose
    $ docker-compose version
    ```

## 3. Flashing CC2531 USB stick at Raspberry Pi

* [Flashing the CC2531 USB stick](https://www.zigbee2mqtt.io/information/flashing_the_cc2531.html)
    ```shell
    $ sudo apt-get install -y dh-autoreconf libusb-1.0-0-dev libboost-all-dev
    $ sudo apt-get install -y git
    $ git clone https://github.com/dashesy/cc-tool.git
    $ cd cc-tool
    $ ./bootstrap
    $ ./configure
    $ make

    $ wget https://github.com/Koenkk/Z-Stack-firmware/raw/master/coordinator/Z-Stack_Home_1.2/bin/default/CC2531_DEFAULT_20201127.zip
    $ unzip CC2531_DEFAULT_20201127.zip

    $ sudo ./cc-tool -e -w CC2531ZNP-Prod.hex

    # Identify the device
    $ ls -l /dev/serial/by-id
    # Identify the group that has access to the device
    $ ls -l /dev/tty*
    # Check the user&group id
    $ id
    ```

## 4. Run `Coziee` containers on Raspberry Pi

* Docker Pull Images at Raspberry Pi
    ```shell
    $ docker pull eclipse-mosquitto
    $ docker pull koenkk/zigbee2mqtt
    $ docker pull oznu/homebridge
    $ docker pull nodered/node-red
    ```
* Connect docker remote host with Context
    ```shell
    $ docker context create rpi --docker "host=ssh://pi@raspberrypi.local"
    $ docker context ls
    $ docker context use rpi
    $ docker ps
    ```
* Start Coziee with docker compose on Raspberry Pi
    ```shell
    $ docker context use rpi
    $ docker compose config
    $ docker compose up -d
    $ docker compose logs -f
    ```

## 5. Configure `mosquitto` container

* Edit `mosquitto.conf`:
    ```shell
    $ docker exec -it mosquitto /bin/sh vi /mosquitto/config/mosquitto.conf
    ```
* Add the following:
    ```
    ...
    listener 1883
    ...
    persistence true
    ...
    persistence_location /mosquitto/data
    ...
    log_dest file /mosquitto/log/mosquitto.log
    ...
    allow_anonymous true
    ...
    ```
* Restart container:
    ```shell
    $ docker compose restart mosquitto
    ```

## 6. Configure `zigbee2mqtt` container

* Edit `configuration.yaml`:
    ```shell
    $ docker exec -it zigbee2mqtt vi /app/data/configuration.yaml
    ```
* add the following:
    ```yaml
    ...
    serial:
        ...
        # Optional: disable LED of the adapter if supported (default: false)
        disable_led: false

    frontend:
        # Optional, default 8080
        port: 8008
        # Optional, default 0.0.0.0
        host: 0.0.0.0
        # Optional, enables authentication, disabled by default
        auth_token: your-secret-token
    ```
* Restart containers:
    ```shell
    $ docker compose restart zigbee2mqtt
    ```
* [Pairing Zigbee Devices](https://www.zigbee2mqtt.io/getting_started/pairing_devices.html)
* Rename devices at: http://raspberrypi.local:8008
    - living_occupancy
    - bedroom_occupancy
    - study_occupancy

## 7. Configure `homebridge` container

* Homebridge UI: http://raspberrypi.local:8581
    - Default User: admin
    - Default password: password
* Setup Users: Setting > User Accounts > Administrator > Edit
* Scan the QR code with the camera on iOS device to add to Apple Home.
* [Install `homebridge-z2m` plugin](https://z2m.dev/install.html)
    - Base topic: zigbee2mqtt (default)
    - Server: mqtt://localhost:1883 (default)
    - Zigbee Identifier / Friendly name: living_occupancy
* [Install `homebridge-platform-wemo` plugin](https://github.com/bwp91/homebridge-platform-wemo/wiki/Installation)
* [Install `homebridge-tplink-smarthome` plugin](https://github.com/plasticrake/homebridge-tplink-smarthome#homebridge-config-ui-x-installation)
* [Install `homebridge-dyson-pure-cool` plugin](https://github.com/lukasroegner/homebridge-dyson-pure-cool#installation)
    - [Retrieve Credentials (Serial & Credentials)](https://github.com/lukasroegner/homebridge-dyson-pure-cool#retrieve-credentials)
* Restart Homebridge to reload plugins configuration

## 8. Configure `nodered` container

* NodeRed: http://raspberrypi.local:1880
