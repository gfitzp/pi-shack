# pi-shack
Steps and notes for installing a GPS-sourced local NTP server and HamClock.

## Components

  - [Raspberry Pi 4](https://www.adafruit.com/product/4296)
  - [Raspberry Pi PoE+ HAT](https://www.adafruit.com/product/5058)
  - [Adafruit Ultimate GPS HAT](https://www.adafruit.com/product/2324)
  - [Raspberry Pi Foundation 7" Display](https://www.adafruit.com/product/2718)
  - [SmartiPi Touch Pro case](https://www.adafruit.com/product/4951)
  - [RAM Mount VESA Plate](https://www.amazon.com/RAM-Mounts-RAM-2461U-75x75mm-Plate/dp/B002UM3PEO)
  - [RAM Mount short arm](https://www.amazon.com/RAM-Mounts-RAM-201U-D-Compatible-Components/dp/B0055BJAV4)
  - [RAM Mount round plate with ball](https://www.amazon.com/RAM-Mounts-Round-Plate-RAM-202U/dp/B0000BVFUC)

I power it with a PoE HAT since as a network time server it's always running, and I didn't have spare power outlets nearby.

I tried running Raspberry Pi OS bullseye, but I can't seem to get it to boot when using the PoE+ HAT. Without the HAT it boots, so one day I may try a different PoE HAT to see if that works. In the meantime I'm using Raspberry Pi OS (Legacy) with desktop.

Don't connect the case's USB-C passthrough cable to the Raspberry Pi board when using the PoE+ HAT; the HAT only expects to get power from the Ethernet jack and not from the Pi.

The SmartiPi case has mounting brackets side-by-side. I mounted the Pi with the PoE HAT on the on the right, and the GPS HAT on the left. I tried using a [ribbon cable](https://www.adafruit.com/product/1988) to connect the two, but there wasn't enough clearance on the PoE HAT for the ribbon cable to connect, so I made my own using [jumper cables](https://www.adafruit.com/product/3141) and [wire housing blocks](https://www.adafruit.com/product/3144).

For the count-down timer I wired the button like the [HamClock user guide](http://www.clearskyinstitute.com/ham/HamClock/#tab-key) suggests (pin 37), but only the green LED (pins 17 and 35) so I don't have a red LED flashing at me when I'm not using the radio. I figure If I don't have a green light, I need to restate my callsign.

I also wired up the satellite alarm (pin 38).

I also added a button to turn the screen on/off on demand (pin 18). The breadboard of the GPS HAT is a great place to solder connections for a capacitor to prevent debouncing. See [screenblanker](https://github.com/gfitzp/screenblanker) for details.

The case has VESA mounting points on the back, so I used the listed RAM Mount accessories to mount the case to my desk so it floats overhead for easy reach.

## Install screenblanker

```
cd ~/Documents &&
git clone https://github.com/gfitzp/screenblanker.git &&
sudo cp screenblanker/screenblanker.service /etc/systemd/system/ &&
sudo systemctl daemon-reload &&
sudo systemctl enable screenblanker.service &&
sudo reboot
```

Add screenblanker schedule to crontab (turns screen on at 9 AM, turns screen off at 11 PM).

```
0  9 * * * /usr/bin/python3 /home/pi/Documents/screenblanker/screenblanker.py noblank >/dev/null 2>&1
0 23 * * * /usr/bin/python3 /home/pi/Documents/screenblanker/screenblanker.py blank   >/dev/null 2>&1
```

Don't forget to disable the Raspberry Pi's built-in screenblanker!

The HamClock has a screenblanking scheduler too, but I wanted the option to easily turn the screen off/on manually and on a schedule.

## Disable login shell over serial, enable serial port hardware

```
sudo raspi-config
```

3 Interface Options > P6 Serial Port


## Improve system performance / make system performance predictable

I believe these set the CPU to a fixed frequency to help with keeping accurate time.

```
echo $(cat /boot/cmdline.txt) "nohz=off" | sudo tee /boot/cmdline.txt &&
echo "performance" | sudo tee /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
```

## Make sure everything's all updated

```
sudo apt-get -y update &&
sudo apt-get -y full-upgrade &&
sudo apt -y auto-remove
```

## Install tools needed for GPS support

```
sudo apt-get -y install gpsd gpsd-clients gpsd-tools pps-tools
```

## Configure GPS, enable PPS, and set other system configuration options

Configure the GPS.

```
sudo tee /etc/default/gpsd > /dev/null <<EOT
# Default settings for the gpsd init script and the hotplug wrapper.

# Start the gpsd daemon automatically at boot time
START_DAEMON="true"

# Use USB hotplugging to add new USB devices automatically to the daemon
USBAUTO="false"

# Devices gpsd should collect to at boot time.
# They need to be read/writeable, either by user gpsd or the group dialout.
DEVICES="/dev/ttyAMA0 /dev/pps0"

# Other options you want to pass to gpsd
GPSD_OPTIONS="-n"
EOT
```

Enable PPS output on the GPIO pin.
```
echo "pps-gpio" | sudo tee -a /etc/modules
```

Enable the serial port, disable Bluetooth for system performance, and customize the PoE hat fan speed temperature triggers. Also [turn off the ethernet jack lights](https://forums.raspberrypi.com/viewtopic.php?t=252049#p1725261) as it's distracting when I'm trying to sleep! :P

```
sudo tee -a /boot/config.txt > /dev/null <<EOT

# Custom configuration starts here

# Enable serial port
enable_uart=1

# Disable bluetooth
dtoverlay=disable-bt

# Enable the PPS signal on GPIO pin
dtoverlay=pps-gpio,gpiopin=4

# PoE Hat Fan Speeds
dtparam=poe_fan_temp0=70000
dtparam=poe_fan_temp0_hyst=2000
dtparam=poe_fan_temp1=80000
dtparam=poe_fan_temp1_hyst=2000
dtparam=poe_fan_temp2=90000
dtparam=poe_fan_temp2_hyst=2000
dtparam=poe_fan_temp3=95000
dtparam=poe_fan_temp3_hyst=5000

[pi4]
# Disable the PWR LED
dtparam=pwr_led_trigger=none
dtparam=pwr_led_activelow=off
# Disable the Activity LED
dtparam=act_led_trigger=none
dtparam=act_led_activelow=off
# Disable ethernet port LEDs
dtparam=eth_led0=4
dtparam=eth_led1=4
EOT
```

## Disable ntp service if present

chronyd will take the place of ntp.

```
sudo systemctl disable ntp
```

## Install chrony

```
sudo apt-get -y install chrony
```

## Configure chrony

```
sudo tee /etc/chrony/chrony.conf > /dev/null <<EOT
# Welcome to the chrony configuration file. See chrony.conf(5) for more
# information about usuable directives.
#pool 2.debian.pool.ntp.org iburst

# This directive specify the location of the file containing ID/key pairs for
# NTP authentication.
keyfile /etc/chrony/chrony.keys

# This directive specify the file into which chronyd will store the rate
# information.
driftfile /var/lib/chrony/chrony.drift

# Uncomment the following line to turn logging on.
#log tracking measurements statistics

# Log files location.
#logdir /var/log/chrony

# Stop bad estimates upsetting machine clock.
maxupdateskew 100.0

# This directive enables kernel synchronisation (every 11 minutes) of the
# real-time clock. Note that it can’t be used along with the 'rtcfile' directive.
rtcsync

# Step the system clock instead of slewing it if the adjustment is larger than
# one second, but only in the first three clock updates.
makestep 1 3

# Custom chrony configuration

# These servers are used when the GPS isn't working for some reason.
# iburst is used on all servers when initially starting to speed up the first clock update
# Poll the local server more frequently (minpoll 3 = 2^3 = at least every 8 seconds)
# Poll remote servers at most every 32 seconds (maxpoll 5 = 2^5 = 32)
# Why? see https://gpsd.gitlab.io/gpsd/gpsd-time-service-howto.html#_arp_is_the_sound_of_your_server_choking

server 192.168.1.1 iburst minpoll 3 maxpoll 5
pool time.apple.com iburst maxpoll 5
pool time.google.com iburst maxpoll 5
pool time.nist.gov iburst maxpoll 5
pool us.pool.ntp.org iburst maxpoll 5

refclock PPS /dev/pps0 refid PPS precision 1e-7 lock NMEA
refclock SHM 0 refid NMEA precision 1e-1 offset 0.541 delay 0.2

# Allow devices to use chrony as their NTP source
allow 127.0.0.1
allow 192.168.0.0/16
allow 100.64.0.0/10

# Enable logging
log tracking measurements statistics
logdir /var/log/chrony
logbanner 21
EOT
```

## Have gpspipe connect once at startup

Sometimes it seems chrony doesn't know about the GPS service until gpsmon is run. Append gpspipe to the `rc.local` startup file and have it output one NMEA line before it exits.

```
sed -i '$ s/exit 0/gpspipe -r -n 1 \&/g' /etc/rc.local ; echo exit 0 >> /etc/rc.local
```

## Start chrony

You may need to experiment in order to determine your particular [offset](https://gpsd.gitlab.io/gpsd/gpsd-time-service-howto.html#_chrony_performance_tuning) and delay values.

```
sudo systemctl restart chronyd &&
watch chronyc tracking
```

## Check chrony status

```
chronyc sources -v
```

## View chrony logs

```
cd /var/logs/chrony
```

## Install hamclock
[HamClock by Elwood Downey, WB0OEW](http://www.clearskyinstitute.com/ham/HamClock/)

```
curl -O http://www.clearskyinstitute.com/ham/HamClock/install-hc-rpi &&
chmod u+x install-hc-rpi &&
./install-hc-rpi
```

### Get the shack's latitude and longitude from cgps

Manually enter in the latitude / longitude from `cgps` into the HamClock setup screen.

It seems if gpsd is enabled in the HamClock, it uses gpsd for both GPS positioning and timekeeping. However, I don't believe gpsd uses the PPS output, so we get more accurate time from our local NTP server; we'll use that instead.

### Set the NTP server to the local chronyd service

Set "NTP?" on Page 2 of the HamClock setup screens to 127.0.0.1; we specifically allow the local machine to poll chrony for the network time in the chrony.conf file.
