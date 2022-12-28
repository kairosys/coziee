# Coziee - Smart Home Automation System

## Table of Contents <!-- omit in toc -->
- [1. Install \& Configure Raspberry Pi OS](#1-install--configure-raspberry-pi-os)
- [2. Install Docker Engine on Raspberry Pi](#2-install-docker-engine-on-raspberry-pi)
- [3. Flashing CC2531 USB stick at Raspberry Pi](#3-flashing-cc2531-usb-stick-at-raspberry-pi)
- [4. Run `coziee` containers on Raspberry Pi](#4-run-coziee-containers-on-raspberry-pi)
- [5. Configure `mosquitto` container](#5-configure-mosquitto-container)
- [6. Configure `zigbee2mqtt` container](#6-configure-zigbee2mqtt-container)
- [7. Configure `homebridge` container](#7-configure-homebridge-container)
- [8. Configure `nodered` container](#8-configure-nodered-container)
- [9. Configure `telegraf` container](#9-configure-telegraf-container)
- [10. Upgrade `coziee` containers on Raspberry Pi](#10-upgrade-coziee-containers-on-raspberry-pi)
- [11. Backup \& Restore Configurations](#11-backup--restore-configurations)
- [12. Backup \& Restore Raspberry Pi SD Card (with `dd` command)](#12-backup--restore-raspberry-pi-sd-card-with-dd-command)
- [13. Backup \& Restore Raspberry Pi SD Card (with `gdd` command)](#13-backup--restore-raspberry-pi-sd-card-with-gdd-command)
- [14. Check System Information of Raspberry Pi](#14-check-system-information-of-raspberry-pi)

## 1. Install & Configure Raspberry Pi OS

* [Download Raspberry Pi OS Lite (64-bit)](https://www.raspberrypi.org/software/operating-systems/)
* [Install Raspberry Pi OS using Raspberry Pi Imager](https://www.raspberrypi.org/software/)
* [Using Raspberry Pi Imager](https://www.raspberrypi.com/documentation/computers/getting-started.html#using-raspberry-pi-imager)
  - `Advanced Options` menu to configure a headless Raspberry Pi
    - Set hostname
    - Enable SSH (Use password authentication)
    - Set username and password
    - Configure wireless LAN
    - Set locale settings
* [Setting up an SSH Server](https://www.raspberrypi.com/documentation/computers/remote-access.html#setting-up-an-ssh-server)
  - SSH Server should be already enabled with the `Advanced Options` at Raspberry Pi Imager.
  - For headless setup, SSH can be enabled by placing a file named ssh, without any extension, onto the boot partition of the SD Card. 
* SSH into Raspberry Pi (Basic Setup)
  ```shell
  $ ssh pi@raspberrypi.local
  # Default password: raspberry
  $ passwd

  # Check locale settings:
  $ locale
  $ locale -a

  # Check the status of WiFi connection
  $ iwconfig
  $ sudo cat /etc/wpa_supplicant/wpa_supplicant.conf

  # Check timezone
  $ timedatectl

  $ sudo raspi-config
  # Update
  # Advanced Options > Expand Filesystem

  # Update package list
  $ sudo apt update
  # Upgrade all installed packages to latest versions
  $ sudo apt list --upgradable
  $ sudo apt full-upgrade

  # Reboot
  $ sudo shutdown -r now
  ```
* To fix the following warning messages about locale settings:
  ```shell
  perl: warning: Setting locale failed.
  perl: warning: Please check that your locale settings:
      LANGUAGE = (unset),
      LC_ALL = (unset),
      LC_CTYPE = "UTF-8",
      LANG = "en_GB.UTF-8"
      are supported and installed on your system.
  perl: warning: Falling back to a fallback locale ("en_GB.UTF-8").
  ```
  - Check locale settings with following command:
    ```shell
    $ locale
    $ locale -a
    ```
  - Edit the file:
    ```shell
    $ sudo vi /etc/default/locale
    ```
  - Update as follow:
    ```
    #  File generated by update-locale
    LANG=en_GB.UTF-8
    LC_CTYPE=en_GB.UTF-8
    LC_MESSAGES=en_GB.UTF-8
    LC_ALL=en_GB.UTF-8
    ```
  - Save and reboot to take effect:
    ```shell
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

## 2. Install Docker Engine on Raspberry Pi

* Install Docker Engine on Raspberry Pi OS
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

  # Verify Docker is installed
  $ docker version

  # Verify Docker Compose is installed
  docker compose version

  # Test with `hello-world` image
  $ docker run --rm hello-world
  $ docker image rm hello-world
  $ docker image rm hello-world
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

## 4. Run `coziee` containers on Raspberry Pi

* Connect docker remote host with context
  ```shell
  $ docker context create coziee --docker "host=ssh://pi@raspberrypi.local"
  $ docker context ls
  $ docker context use coziee
  $ docker ps
  ```
* Docker pull images at Raspberry Pi
  ```shell
  $ docker pull eclipse-mosquitto
  $ docker pull koenkk/zigbee2mqtt
  $ docker pull oznu/homebridge
  $ docker pull nodered/node-red
  $ docker pull telegraf
  ```
* Start `coziee` with docker compose on Raspberry Pi
  ```shell
  $ cd ~/Workspaces/coziee
  $ docker context use coziee
  $ docker compose config
  $ docker compose up -d
  $ docker compose logs -f
  ```

## 5. Configure `mosquitto` container

* Edit `mosquitto.conf`:
  ```shell
  $ docker exec -it mosquitto mosquitto_passwd -c /mosquitto/config/passwd coziee
  $ docker exec -it mosquitto cat /mosquitto/config/passwd
  $ docker exec -it mosquitto chown mosquitto:mosquitto /mosquitto/config/passwd
  $ docker exec -it mosquitto ls -latr /mosquitto/config
  $ docker exec -it mosquitto vi /mosquitto/config/mosquitto.conf
  ```
* Update as follow:
  ```conf
  ...
  listener 1883
  ...
  persistence true
  ...
  persistence_location /mosquitto/data
  ...
  log_dest stderr
  log_dest file /mosquitto/log/mosquitto.log
  ...
  password_file /mosquitto/config/passwd
  ...
  ```
* Restart container:
  ```shell
  $ docker compose restart mosquitto
  $ docker compose logs -f mosquitto
  ```

## 6. Configure `zigbee2mqtt` container

* Edit `configuration.yaml`:
  ```shell
  # Edit directly at zigbee2mqtt container
  $ docker exec -it zigbee2mqtt vi /app/data/configuration.yaml

  # Edit indirectly with busybox container
  $ docker run --volumes-from zigbee2mqtt -it busybox vi /app/data/configuration.yaml
  ```
* Update [Configuration](https://www.zigbee2mqtt.io/guide/configuration/) as follow:
  ```yaml
  ...
  mqtt:
    ...
    # MQTT server URL
    server: mqtt://mosquitto:1883
    # MQTT server authentication, uncomment if required:
    user: my_user
    password: my_password
  
  serial:
    # Location of CC2531 USB sniffer
    port: /dev/ttyACM0
    # Optional: disable LED of the adapter if supported (default: false)
    disable_led: true

  frontend: true
  ```
* Restart container:
  ```shell
  $ docker compose restart zigbee2mqtt
  $ docker compose logs -f zigbee2mqtt
  ```
* [Pairing Zigbee Devices](https://www.zigbee2mqtt.io/getting_started/pairing_devices.html)
* Rename devices at: http://raspberrypi.local:8008
  - living_occupancy
  - study_occupancy
  - bedroom_occupancy
  - outdoor_weather

## 7. Configure `homebridge` container

* Homebridge UI: http://raspberrypi.local:8581
  - Create user account with username and password
* Edit Users: Setting > User Accounts > Administrator > Edit
* [Install `homebridge-z2m` plugin](https://z2m.dev/install.html)
  - Base topic: zigbee2mqtt (default)
  - Server: mqtt://localhost:1883 (default)
  - Username: coziee
  - Password: [password]
* [Install `homebridge-dummy` plugin](https://github.com/nfarina/homebridge-dummy)
  - Name: Sensor Light Switch
  - Stateful: Checked
  - Reverse: Checked
* [Install `homebridge-wemo` plugin](https://github.com/bwp91/homebridge-wemo/wiki/Installation)
* [Install `homebridge-tplink-smarthome` plugin](https://github.com/plasticrake/homebridge-tplink-smarthome#homebridge-config-ui-x-installation)
* [Install `homebridge-dyson-pure-cool` plugin](https://github.com/lukasroegner/homebridge-dyson-pure-cool#installation)
* Restart Homebridge to reload plugins configuration
* [Retrieve Credentials (Serial & Credentials) for Dyson device](https://github.com/lukasroegner/homebridge-dyson-pure-cool#retrieve-credentials)
    - Open a browser and navigate to: http://raspberrypi.local:48000
* Scan the QR code with the camera on iOS device to add to Apple Home.
* Setting > Backup / Restore > Download Backup Archive

## 8. Configure `nodered` container

* Generating the password hash
  ```shell
  $ docker exec -it nodered node-red admin hash-pw
  $ docker exec -it nodered node -e "console.log(require('bcryptjs').hashSync(process.argv[1], 8));" your-password-here
  ```
* Edit `settings.js`:
  ```shell
  $ docker exec -it nodered vi /data/settings.js
  ```
* Update as follow:
  ```js
  adminAuth: {
      type: "credentials",
      users: [{
          username: "admin",
          password: "$PASSWORD_HASH",
          permissions: "*"
      }]
  },
  ```
* Restart container:
  ```shell
  $ docker compose restart nodered
  $ docker compose logs -f nodered
  ```
* NodeRed Editor: http://raspberrypi.local:1880
* Settings > Palette > Install
  - node-red-node-openweathermap
  - node-red-contrib-homebridge-automation
  - node-red-contrib-huemagic
  - node-red-contrib-chronos
  - node-red-contrib-telegrambot

## 9. Configure `telegraf` container

* Edit `telegraf.conf`:
  ```shell
  $ docker run --rm -it -v telegraf-vol:/etc/telegraf busybox vi /etc/telegraf/telegraf.conf
  ```
* Update as follow:
  ```conf
  ...
  # Configuration for sending metrics to InfluxDB
  # [[outputs.influxdb]]
  ...
  # Configuration for sending metrics to InfluxDB
  [[outputs.influxdb_v2]]
    ## The URLs of the InfluxDB cluster nodes.
    urls = ["http://192.168.1.2:8086"]

    ## Token for authentication.
    token = "$TOKEN"

    ## Organization is the name of the organization you wish to write to; must exist.
    organization = "Homelab"

    ## Destination bucket to write into.
    bucket = "telegraf"

    namedrop = ["sensor"]
  ...
  [[outputs.influxdb_v2]]
    ## The URLs of the InfluxDB cluster nodes.
    urls = ["http://192.168.1.2:8086"]

    ## Token for authentication.
    token = "$TOKEN"

    ## Organization is the name of the organization you wish to write to; must exist.
    organization = "Homelab"

    ## Destination bucket to write into.
    bucket = "sensors"

    namepass = ["sensor"]
  ...
  # Read metrics about disk usage by mount point
  [[inputs.disk]]
  ...
    ## Ignore mount points by filesystem type.
    ignore_fs = ["tmpfs", "devtmpfs", "devfs", "iso9660", "overlay", "aufs", "squashfs", "nsfs"]
  ...
  # Read metrics from MQTT topic(s)
  [[inputs.mqtt_consumer]]
    
    name_override = "sensor"

    ## Broker URLs for the MQTT server or cluster.
    servers = ["tcp://mosquitto:1883"]

    ## Topics that will be subscribed to.
    topics = [
      "telegraf/host01/cpu",
      "telegraf/+/mem",
      "sensors/#",
    ]
  ...
    ## Username and password to connect MQTT server.
    username = "coziee"
    password = "[password]"
  ...
    ## Data format to consume.
    data_format = "value"
    data_type = "float"
  ...
  ```
* Restart container:
  ```shell
  $ docker compose restart telegraf
  $ docker compose logs -f telegraf
  ```

## 10. Upgrade `coziee` containers on Raspberry Pi

* Upgrade `coziee` containers on Raspberry Pi:
  ```shell
  $ cd ~/Workspaces/coziee
  $ docker context use coziee
  $ docker compose pull
  $ docker compose up -d
  $ docker compose logs -f
  $ docker image prune -a
  ```

## 11. Backup & Restore Configurations

* Backup configuration files:
  ```shell
  $ cd ~/Workspaces/coziee/backup
  $ docker cp -a mosquitto:/mosquitto/config/mosquitto.conf .
  $ docker cp -a mosquitto:/mosquitto/config/passwd .
  $ docker cp -a zigbee2mqtt:/app/data/configuration.yaml .
  $ docker cp -a homebridge:/homebridge/config.json .
  $ docker cp -a nodered:/data/settings.js .
  $ docker cp -a telegraf:/etc/telegraf/telegraf.conf .
  ```
* Restore configuration files:
  ```shell
  $ cd ~/Workspaces/coziee/backup
  $ docker cp -a mosquitto.conf mosquitto:/mosquitto/config
  $ docker cp -a passwd mosquitto:/mosquitto/config
  $ docker cp -a configuration.yaml zigbee2mqtt:/app/data
  $ docker cp -a config.json homebridge:/homebridge
  $ docker cp -a settings.js nodered:/data
  $ docker cp -a telegraf.conf telegraf:/etc/telegraf
  ```
docker
## 12. Backup & Restore Raspberry Pi SD Card (with `dd` command)

* Install [Pipe Viewer](http://www.ivarch.com/programs/pv.shtml) to track the progress of `dd` command.
  ```shell
  $ brew install pv
  ```
* Identify SD card name:
  ```shell
  $ diskutil list
  ```
* Use `dd` command to backup SD card via raw disk `/dev/disk4` and pipe through `pv` command with the provide of input size:
  ```shell
  $ cd ~/Downloads
  $ sudo dd if=/dev/disk4 bs=1m  | pv -s 16G | sudo dd of=rpi_arm64_coziee_"`date +"%Y-%m-%d"`".dmg bs=1m
  ```
* Alternatively for macOS, pressing `CTRL+T` to track the progress of `dd` command.
  ```shell
  load: 2.11  cmd: dd 6323 uninterruptible 0.00u 1.85s
  7798+0 records in
  7798+0 records out
  8176795648 bytes transferred in 785.410470 secs (10410856 bytes/sec)
  ```
* Use `dd` command to restore SD card:
  ```shell
  $ diskutil unmountDisk /dev/disk5
  $ sudo dd if=rpi_arm64_coziee_YYYY-MM-DD.dmg bs=1m | pv -s 16G | sudo dd of=/dev/disk4 bs=1m
  ```

## 13. Backup & Restore Raspberry Pi SD Card (with `gdd` command)

* Install `coreutils` with a version of `dd` command that supports the status option.
  ```shell
  $ brew install coreutils
  ```
* Identify SD card name:
  ```shell
  $ diskutil list
  ```
* Use `gdd` command to backup SD card via raw disk `/dev/disk4` with status option:
  ```shell
  $ cd ~/Downloads
  $ sudo gdd if=/dev/disk4 of=rpi_arm64_coziee_"`date +"%Y-%m-%d"`".dmg bs=1M status=progress
  ```
* Use `gdd` command to restore SD card:
  ```shell
  $ diskutil unmountDisk /dev/disk5
  $ sudo dd if=rpi_arm64_coziee_YYYY-MM-DD.dmg of=/dev/disk4 bs=1M status=progress
  ```

## 14. Check System Information of Raspberry Pi 

* Print system information
  ```shell
  $ uname -a
  ```
* Print CPU infromation and hardware model
  ```shell
  $ cat /proc/cpuinfo
  ```
* Display Linux processes:
  ```shell
  $ top -i
  ```
* Display amount of free and used memory in the system:
  ```shell
  $ free -h
  ```
* Report file system disk space usage
  ```shell
  $ df -h
  ```
