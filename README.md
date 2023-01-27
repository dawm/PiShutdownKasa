# PiShutdownKasa
This method allows you to turn off a Kasa smart device upon shutdown of your raspberry pi. This guide has been tested on the KP115 smartplug, a list of compatible devices (that have the timer function) is listed below.<br><br>
The method used here was taken from [this discussion](https://github.com/python-kasa/python-kasa/discussions/284) on the [python-kasa](https://github.com/python-kasa/python-kasa) github. I'm no expert but this is the method I was able to figure out that worked for me.<p>
The reason needed to create this for my setup is because my raspberry pi controls a 3D printer and when I shutdown the pi I then have to turn off the printer manually *(turning the power off on a raspberry pi without properly shutting down can lead to SD card corruption)*, instead of purchasing a relay board and configuring that into the setup I thought I could use my Kasa smartplug to do the job (its essentially a wifi relay).

**Compatible devices:**
RE270K , HS100P3 , EP40 , HS107 , HS105 KIT , HS103 , HS300 , KP115 , HS105 , KD110 , HS110 KIT , HS220 , HS200 , HS100 , RE370K , KP100 , KP400 , KP125 , KP200 , HS100 KIT , KP105 , HS210 KIT , KP303 , HS110 , HS210 - [Source](https://www.tp-link.com/us/support/faq/947/)
### Prerequisite
- Your Kasa smartplug should already be setup and working on your network. Preferably with a static IP set in your router settings.
- There is an alias feature where you can name your device so that you don't have to use the IP address but I will not be using that method that in this guide. But if you want to you can set an alias with `kasa --host 192.168.xx.xx alias AliasNameHere`, then after it is set you can replace `--host 192.168.xx.xx` in this guide with `--alias AliasNameHere`.
### Step 1
[Install python-kasa](https://github.com/python-kasa/python-kasa#getting-started)
### Step 2
>To search your network for the smart plug, use the command: `kasa discover`
```
$ kasa discover
Discovering devices on 255.255.255.255 for 3 seconds
== MyOutlet - KP115(US) ==
        Host: 192.168.xx.xx
        Device state: OFF

        == Generic information ==
        Time:         2023-01-07 01:01:01 (tz: {'index': 18, 'err_code': 0}
        Hardware:     1.0
        Software:     1.0.18 Build 210910 Rel.141202
        MAC (rssi):   XX:XX:XX:XX:XX:XX (-26)
        Location:     {'latitude': XX.XXXX, 'longitude': XX.XXXX}

        == Device specific information ==
        LED state: True
        On since: None

        == Current State ==
        <EmeterStatus power=0.0 voltage=121.682 current=0.0 total=0.151>

        == Modules ==
        + <Module Schedule (schedule) for 192.168.xx.xx>
        + <Module Usage (schedule) for 192.168.xx.xx>
        + <Module Antitheft (anti_theft) for 192.168.xx.xx>
        + <Module Time (time) for 192.168.xx.xx>
        + <Module Cloud (cnCloud) for 192.168.xx.xx>
        + <Module Emeter (emeter) for 192.168.xx.xx>
```
- Configure a countdown rule on the outlet (KP115 only allows 1 countdown rule)
>**First we get a list of the current rule(s) on our specific device**
<br>Use the command `kasa --host 192.168.xx.xx raw-command count_down get_rules`, replace `192.168.xx.xx` with the IP of the device.
```
$ kasa --host 192.168.xx.xx raw-command count_down get_rules
No --type defined, discovering..
{'rule_list': [{'id': 'A553DCB49A54401399DB130F9E589D5C', 'name': 'turnoff45s', 'enable': 0, 'delay': 0, 'act': 0, 'remain': 0}]}
```
>**If no rules are found we have to create one, otherwise skip to editing the rule**
```
$ kasa --host 192.168.xx.xx raw-command count_down add_rule '{"enable": 0, "delay": 45, "act": 0, "name": "turnoff45s"}'
No --strip nor --bulb nor --plug given, discovering..
{'id': 'A553DCB49A54401399DB130F9E589D5C'}
```
`id:` is the rule ID given upon creation of a new rule.<br>
`enable:` determines if the rule is enabled, `0 = disabled`, `1 = enabled`.<br>
`delay:` is how long in seconds to run the timer. Ex: `delay: 45` would be 45 second countdown.<br>
`act:` determines if the rule turns on the outlet or turns it off, `0 = OFF`, `1 = ON`.<br>
`name:` allows you to give the rule a name.

>**If we already had a rule then we edit the rule**<br>
```
$ kasa --host 192.168.xx.xx raw-command count_down edit_rule '{"id":"A553DCB49A54401399DB130F9E589D5C","enable": 0, "delay": 45, "act": 0, "name": "turnoff45s"}'
```
### Step 3
>Now we create a script that will update the rule to enable the timer
```
$ cd ~
$ nano turnOffOutlet.sh
```
>Contents of **turnOffOutlet.sh**
```
#!/bin/bash

#Check to see if the outlet is currently turned on, no need to turn it off otherwise.
if kasa --host 192.168.xx.xx state | grep -q 'Device state: ON'; then
        kasa --host 192.168.xx.xx raw-command count_down edit_rule '{"id":"A553DCB49A54401399DB130F9E589D5C","enable": 1, "delay": 45, "act": 0, "name": "turnoff45s"}'
fi
```
>Set permissions and move the script file to `/usr/local/bin`
```
$ chmod u+x ~/turnOffOutlet.sh
$ sudo mv ~/turnOffOutlet.sh /usr/local/bin/
```
> Now would be a good time to test our script, turn on the outlet and run the script. The outlet should turn off after the amount of seconds you set in the delay field.
```
$ /usr/local/bin/turnOffOutlet.sh
```
### Step 4
>Now we create a system service to run our script upon shutdown
```
$ sudo nano /etc/systemd/system/turnoffoutlet.service
```
>Contents of /etc/systemd/system/**turnoffoutlet.service**
```
[Unit]
Description="KP115 outlet control"
Requires=network.target network-online.target
Before=shutdown.target
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
RemainAfterExit=true
ExecStart=/bin/true
ExecStop=/usr/local/bin/turnOffOutlet.sh

[Install]
WantedBy=multi-user.target
```
>Run the following commands to enable the new service
```
$ sudo systemctl daemon-reload
$ sudo systemctl enable turnoffoutlet.service
$ sudo systemctl start turnoffoutlet
```
### And thats it, you're done.
>Turn on the outlet and try shutting down your pi.<br>
If all goes well the outlet will turn off after the delay time you set.
```
$ sudo shutdown -now
```
