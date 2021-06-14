# Coziee - Home Automation System

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

* Edit `mosquitto.conf` within `mosquitto` container
    ```shell
    $ docker exec -it mosquitto /bin/sh

    $ vi /mosquitto/config/mosquitto.conf
    # listener 1883
    # persistence true
    # persistence_location /mosquitto/data
    # log_dest file /mosquitto/log/mosquitto.log
    # allow_anonymous true
    $ exit

    $ docker compose restart
    ```

* Configuration UI:
    - Homebridge UI: http://raspberrypi.local:8581
    - NodeRed: http://raspberrypi.local:1880/
