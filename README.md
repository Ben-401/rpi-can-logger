# Raspberry PI CAN Bus Logger

This project provides code for logging CAN Bus data with a Raspberry Pi. It additionally also logs GPS data. All of this data is stored on an SD card and intended to be used in a running .

## Features

* Logs CAN bus from either OBD2 or Tesla vehicles
* Logs FMS
* Logs GPS
* Can operate in polling and sniffing mode
* Stores data on SD card. Can be configured to automatically upload to FTP or web service when connected to WiFi or 4G internet.
* Can be powered entirely from power provided by the OBD port in your vehicle!  You can also wire it into your fuse box or cigarette lighter to prevent it being powered permanently and draining your battery.
* Accompanying Bluetooth App to visualise your data while you drive (coming soon)
* Web based data visualiser (coming soon)

## Parts

The following parts are used:

* Raspberry PI 3 Model B
* [PiCAN CAN-Bus board](http://skpang.co.uk/catalog/pican2-duo-canbus-board-for-raspberry-pi-23-p-1480.html) or equivalent PiCAN product with 1 or 2 CAN buses
* [GPS Receiver](https://www.dfrobot.com/product-1103.html)
* [DC-DC Converter](https://www.digikey.com.au/products/en?keywords=1597-1243-ND)


You'll need to do some soldering to make the connector from your OBD port or [Tesla port](http://au.rs-online.com/web/p/pcb-connector-housings/7201162/?searchTerm=720-1162&relevancy-data=636F3D3126696E3D4931384E525353746F636B4E756D626572266C753D656E266D6D3D6D61746368616C6C26706D3D5E285C647B362C377D5B4161426250705D297C285C647B337D5B5C732D2F255C2E2C5D5C647B332C347D5B4161426250705D3F292426706F3D3126736E3D592673743D52535F53544F434B5F4E554D4245522677633D4E4F4E45267573743D3732302D31313632267374613D3732303131363226)to your PiCAN.

If you want WiFi to work with the PiCAN2 shield attached, you'll need to unsolder the GPIO pins and drop them to the bottom and reattach the shield.

Better description of all necessary parts (coming soon).

## Full Setup

1. Download the latest raspbian  lite image from here: https://raspberrypi.org/downloads/raspbian
2. Insert your SD card into your computer
3. Use your preferred method to put the rasbpian image onto your machine. On linux:
````bash
wget https://downloads.raspberrypi.org/raspbian_lite_latest
tar -xvf  raspbian_lite_latest
# the if argument might be different
dd if=2017-09-07-raspbian_stretch-lite.img of=/dev/sdb bs=4M conv=fsync status=progress
````
4. Unmount your SD card, and plug it into your raspberry pi
5. Run the following commands after logging in (default username is `pi`, password is `raspberry`) and configuring 
wifi by putting your settings in `/etc/wpa_supplicant/wpa_supplicant.conf` (you will need to restart the wifi to have
 the settings take effect by 
running `sudo service networking restart`):
````bash
sudo apt update
sudo apt install git make build-essential libssl-dev zlib1g-dev libbz2-dev libreadline-dev libsqlite3-dev wget curl llvm libncurses5-dev xz-utils bluez python-bluez pi-bluetooth
curl -L https://raw.githubusercontent.com/pyenv/pyenv-installer/master/bin/pyenv-installer | bash
env PYTHON_CONFIGURE_OPTS="--enable-shared" pyenv install 3.6.2
git clone https://github.com/JonnoFTW/rpi-can-logger.git
````
### Install Rpi-Logger

1. Depending on which configuration file you want to use, edit the argument in the `systemd/rpi-logger.service` file 
 on line 12, to be the configuration you want
1. Run 
```bash
sudo pip3 install -r requirements.txt
sudo python3 setup.py install
```
 to install everything. You'll need root access if you want it to be installed a service that runs on startup.
1. If you want to change your hostname, run the `pairable.py` script in the `systemd` folder
1. Enable UART on your RPI (for the GPS) and CAN (skip the second `dtoverlay` line if your CAN shield only has 1 input)
 for the CAN shield by adding these lines to `/boot/config.txt`:
```
enable_uart=1
dtparam=spi=on
dtoverlay=mcp2515-can0,oscillator=16000000,interrupt=25
dtoverlay=mcp2515-can1,oscillator=16000000,interrupt=24
dtoverlay=spi-bcm2835
```
4. In order to stop the RPI from asking your serial ports `/ttyS0` to log on, change `/boot/cmdline.txt` and remove:
```
console=serial0,baudrate=115200
```
5. Add these lines to your `/etc/network/interfaces` file (set it to 250000 if you are using FMS and skip `can1` if 
you only have 1 CAN port):
```
auto can0
iface can0 inet manual
    pre-up /sbin/ip link set can0 type can bitrate 500000 triple-sampling on
    up /sbin/ifconfig can0 up
    down /sbin/ifconfig can0 down

auto can1
iface can1 inet manual
    pre-up /sbin/ip link set can1 type can bitrate 500000 triple-sampling on
    up /sbin/ifconfig can1 up
    down /sbin/ifconfig can1 down

```

5. The logging and file upload service will now run on startup. By default it will use: [example_fms_logging.yaml](https://github.com/JonnoFTW/rpi-can-logger/blob/master/example_fms_logging.yaml).
6. To setup uploading of files, you will need to create a `mongo_conf.yaml` file in the project directory:
```yaml
log_dir: ~/log/can-log/
keys:
 - vid_key_1
 - vid_key_2
api_url: http://url.to/api/ # the api on the end is important
```
  
## Configuration
RPI-CAN-Logger is highly configurable and supports nearly all standard OBD-2 PIDs and the currently understood frames from Tesla as described in [this document].
### Configuring CAN Logging

We currently support 3 forms of logging  (with CAN FMS and CAN FD to come soon):

* [Sniffing OBD](https://github.com/JonnoFTW/rpi-can-logger/blob/master/example_obd_sniffing_conf.yaml)
* [Querying OBD](https://github.com/JonnoFTW/rpi-can-logger/blob/master/example_obd_querying_conf.yaml)
* [Sniffing Tesla](https://github.com/JonnoFTW/rpi-can-logger/blob/master/example_tesla_conf.yaml)

Here we will examine the various configuration options:

```yaml
interface: socketcan_native # can bus driver
channel: can1 # which can bus to use
log-messages: /home/pi/log/can-log/messages/ # location of debug messages
log-folder: /home/pi/log/can-log/ # location of log files
log-size: 32 # maximum size in MB before log file is rotated 
log-pids: # the pids we want to log, refer to rpi_can_logger/logger/obd_pids.py,  tesla_pids.py fms_pids.py outlander_pid
  - PID_ENGINE_LOAD
  - PID_THROTTLE
  - PID_INTAKE_MAP
  - PID_RPM
  - PID_SPEED
  - PID_MAF_FLOW
log-trigger: PID_MAF_FLOW # when this PID is seen, return the buffer in current state (only works in sniffing mode)
disable-gps: false # enable/disable GPS logging
gps-port: /dev/ttyS0 # serial port for GPS device
tesla: false # indicates whether or not we are logging a tesla vehicle
sniffing: false # indicates that we are sniffing
log-bluetooth: true # whether or not we log to bluetooth
bluetooth-pass: super_secret_password # the password required to stream the bluetooth data
log-level: warning # log level (warning or debug)
vid: vehicle_identifier # a unique way to identify this vehicle
fms: false # are we logging FMS? (Bus and Truck only)
verbose: true # give verbose message output on stdout
```

### Configuring Data Upload

In the root directory of this project create a file called: `mongo_conf.yaml`, it should look like this:

```yaml
log_dir: /home/pi/log/can-log/
pid-file: ~/log/can-log.pid
api_url: 'https://url.to.server.com/api/'
keys:
    - vid_key_1
    - vid_key_2
```
The keys are the API keys for each vehicle that this logger will log for.

### Cloning SD Cards

Because we're deploying to a lot of these devices, you'll need to make an image after setting everything up on your SD
card. Once you're done, plug your SD card into another computer and run:

`dd of=logger.img if=/dev/sdb bs=4M conv=fsync status=progress`

Once that's finished you'll have a file called `logger.img` on your machine, insert a new card and run:

`dd if=logger.img of=/dev/sdb bs=4M conv=fsync status=progress`

This should clone the SD card assuming they're exactly the same. If the cards are different sizes (ie. the new card is 
**LARGER**), run `raspi-config` and resize the partition.

If the target card is smaller (and assuming the amount of data used on the image is less than the target SD card size),
 then you will need to resize the partition.
 
 
After you've done all that, boot up your new device with a clone SD card and modify the following:

 * `/etc/hostname`
 * `/etc/hosts` 
To use a unique hostname and restart the device. You can easily do this by running the `systemd/pairable.py` script like this:

```bash
./systemd/pariable.py rpi-logger-12345
sudo reboot
```

Where `12345` is the vehicle identifier. In order to connect via the bluetooth app, the device hostname must start with `rpi-logger`
 
You'll also probably need to pair the bluetooth with your phone, run:

```bash
sudo bluetoothctl
discoverable on
pairable on
```
Initiate pairing on your phone. Then run:
```bash
discoverable off
pairable off
quit
```

