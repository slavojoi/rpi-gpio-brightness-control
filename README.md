# rpi-gpio-brightness-control
This is how to setup brightness control of RPI Official Touchscreen 7" display via GPIO Trigger (GPIO PIN 3 = Ignition and GPIO 15 = Car Headlights)

1. create new systemd service that will starting on boot of RPI   
`sudo nano /etc/systemd/system/illumination.service`
2. paste this in there
```
[Unit]    
   Description=RPI Display GPIO Brightness Trigger Service   
   ConditionPathExists=/opt/illumination/service_illumination.sh   
   
    [Service]   
   Type=simple   
   ExecStart=/opt/illumination/service_illumination.sh   
   RemainAfterExit=yes   
   
    [Install]   
   WantedBy=multi-user.target
```
3. create illumination folder in opt
`sudo mkdir /opt/illumination`
4. create variable file   
`sudo nano /opt/illumination/illumination_env.sh`
5. paste this
```
## Hardware ###
# The hardware pins can be completly disabled with the global flag.

# Global Flag (enables / disables gpio usage excluding device connected
# trigger gpio and ignition based shutdown!)
ENABLE_GPIO=1


# GPIO Trigger for Day/Night
# GPIO wich triggers Day (open gpio)/Night (closed to gnd) of GUI
# To disable set to 0
# If enabled it overrides lightsensor!
DAYNIGHT_PIN=15
IGNITION_PIN=3

### Screen ###
# Brightness related stuff
# brightness file (default for rpi display: /sys/class/backlight/rpi_backlight/brightness)
BRIGHTNESS_FILE=/sys/class/backlight/rpi_backlight/brightness

# brightness values
BR_MIN=30
BR_MAX=255
BR_STEP=25
BR_DAY=255
BR_NIGHT=30
BR_IGNITION=15
```
6. create script
`sudo nano /opt/illumination/service_illumination.sh`
7. paste this
```
#!/bin/bash

source /opt/illumination/illumination_env.sh

# check gpio pin if activated
if [ $DAYNIGHT_PIN -ne 0 ]; then
    while true; do
		# turn backlight down/up based on car ignition signal (I turn my backlight off)
        if [ $IGNITION_PIN -ne 0 ]; then
#            IGNITION_GPIO=`gpio -g read $IGNITION_PIN`
			IGNITION_GPIO=$(raspi-gpio get $IGNITION_PIN)
			IGNITION_GPIO=${IGNITION_GPIO##* level=}
			IGNITION_GPIO=${IGNITION_GPIO%% *}
			if [ $IGNITION_GPIO -eq 0 ]; then
				if [ ! -f /tmp/ignition_dim_enabled ]; then
					touch /tmp/ignition_dim_enabled
					echo $BR_IGNITION > $BRIGHTNESS_FILE
				fi
			else
				if [ -f /tmp/ignition_dim_enabled ]; then
					rm /tmp/ignition_dim_enabled >/dev/null 2>&1
					if [ -f /tmp/night_mode_enabled ]; then
						echo $BR_NIGHT > $BRIGHTNESS_FILE
					else
						echo $BR_DAY > $BRIGHTNESS_FILE
					fi
				fi
			fi
		fi
		# cycle day/night based on $DAYNIGHT_GPIO
        if [ ! -f /tmp/ignition_dim_enabled ]; then # skip day/night cycle if ignition_dim_enabled
            if [ ! -f /tmp/illumination_brightness_active ]; then
                touch /tmp/illumination_brightness_active
            fi
#            DAYNIGHT_GPIO=`gpio -g read $DAYNIGHT_PIN`
			DAYNIGHT_GPIO=$(raspi-gpio get $DAYNIGHT_PIN)
			DAYNIGHT_GPIO=${DAYNIGHT_GPIO##* level=}
			DAYNIGHT_GPIO=${DAYNIGHT_GPIO%% *}
            if [ $DAYNIGHT_GPIO -eq 1 ]; then
#                if [ ! -f /tmp/night_mode_enabled ]; then
#                    touch /tmp/night_mode_enabled
                    echo $BR_NIGHT > $BRIGHTNESS_FILE
#                fi
            else
#                if [ -f /tmp/night_mode_enabled ]; then
#                    rm /tmp/night_mode_enabled >/dev/null 2>&1
                    echo $BR_DAY > $BRIGHTNESS_FILE
#                fi
            fi
        else
            if [ -f /tmp/illumination_brightness_active ]; then
                rm /tmp/illumination_brightness_active >/dev/null 2>&1
            fi
        fi		
        sleep 1
    done
fi

exit 0
```
8. Make sure the permissions on the script and the service file are correct. They should be owned by root and the script should be executable.   
9. `sudo chmod 744 /opt/illumination/service_illumination.sh`
10. `sudo chmod 744 /opt/illumination/illumination_env.sh`
11. `sudo chmod 644 /etc/systemd/system/illumination.service`
12. Enable the service on boot `sudo systemctl enable illumination.service`
