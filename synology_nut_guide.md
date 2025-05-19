# Connect Synology NAS to a Linux NUT Server (UPS Monitoring)

Use this guide to configure your Synology NAS to monitor a UPS managed
by a NUT server on a Linux machine.

This document makes the following assumptions:
 * A cyberpower UPS
 * Connected via a USB cable to a Linux box of some sort at 10.10.0.17
 * Network UPS Tools are installed ```apt install nut```
 * A NUT client device at 10.10.0.18

---

## On the Linux NUT Server (e.g., 10.10.0.17)

### 1. `/etc/nut/ups.conf`

> **UPS name must be `[ups]` — Synology requires this.**

```
# lsusb output
# Bus 001 Device 003: ID 0764:0601 Cyber Power System, Inc. PR1500LCDRT2U UPS
# Use the ID information vendorid:productid
# Cyberpower UPS's seem to need a polling interval less than 20 seconds
# otherwise the monitoring daemon fails with log messages like
#   upsmon[3888210]: Poll UPS [ups@localhost] failed - Data stale
#   upsmon[3888210]: Communications with UPS ups@localhost lost  

[ups]
    driver = usbhid-ups
    port = auto
    vendorid = 0764
    productid = 0601
    pollinterval = 15   # https://nmaggioni.xyz/2017/03/14/NUT-CyberPower-UPS/
    desc = "CyberPower CP850PFCLCD"
```

---

### 2. `/etc/nut/upsd.conf`

Allow connections from Synology (replace IP if needed):

```
LISTEN 0.0.0.0 3493
```
---

### 3. `/etc/nut/upsd.users`

Create the username and password REQUIRED by Synology; e.g., monuser /
secret

```
[monuser]
    password = secret
    upsmon slave       # or 'upsmon secondary' if using NUT 2.8+
```

---

### 4. `/etc/nut/nut.conf`

```
MODE=standalone
```

---

### 5. `/etc/nut/upsmon.conf`
```
MONITOR ups@localhost 1 upsmon secret master # or 'primary' if using NUT 2.8+
```

---

### 6. Restart NUT Services

```
sudo systemctl restart nut-server nut-driver nut-monitor
```

---

### 7. (Optional) Open Firewall Port
This example opens fw for traffic from the device at 10.10.0.18

```
sudo ufw allow from 10.10.0.18 to any port 3493 proto tcp
```

---

##  On Synology NAS

1. Go to **Control Panel** → **Hardware & Power** → **UPS** tab  
2. Enable **UPS support**  
3. Choose:
   - **UPS Type**: `Synology UPS Server`
   - **UPS Server IP**: `10.10.0.17`
4. Click **Device Information** to verify connection
5. Set shutdown behavior as desired

---

## Test (Optional)

From Synology (via SSH):

```
upsc ups@10.10.0.17
```

Expected output:

```
Init SSL without certificate database
battery.charge: 100
battery.charge.low: 10
battery.charge.warning: 20
battery.mfr.date: CPS
battery.runtime: 1150
battery.runtime.low: 300
battery.type: PbAcid
battery.voltage: 13.6
battery.voltage.nominal: 12
...
```

## Monit (Optional)
From time to time the usb driver fails on the linux box and the
Internet mind isn't certain why. Therefore monit can be added to the
configuration to look for problems and fix them.

1. Create a script to look at the last 5 minutes of journalctl data
and put it into /usr/local/bin/check_nut_device_log.sh
```
#!/bin/bash

# Check for the error in the last 5 minutes of logs
journalctl -u nut-server --since "5 minutes ago" | grep -q "Data for UPS \[ups\] is stale - check driver"

if [ $? -eq 0 ]; then
  # Problem found
  exit 1
else
  # No issue
  exit 0
fi
```
Make it executable
```chmod +x /usr/local/bin/check_nut_device_log.sh```

2. Create a monit script in
/etc/monit/conf-available/nut-driver-monitor
```
CHECK PROGRAM nut_driver_log_check WITH PATH "/usr/local/bin/check_nut_device_log.sh"
    EVERY 1 CYCLE
    IF STATUS != 0 THEN RESTART
    START PROGRAM = "/bin/systemctl start nut-driver.service"
    STOP PROGRAM = "/bin/systemctl stop nut-driver.service"
```
link the script
```
cd /etc/monit/conf-enabled
ln -s ../conf-available/nut-driver-monitor .
```
restart monit
```
service monit stop
service monit start
```
