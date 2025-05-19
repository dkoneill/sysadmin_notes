# Connect Synology NAS to a Linux NUT Server (UPS Monitoring)

Use this guide to configure your Synology NAS to monitor a UPS managed by a NUT server on a Linux machine.

---

## On the Linux NUT Server (e.g., 10.10.0.17)

### 1. `/etc/nut/ups.conf`

> **UPS name must be `[ups]` — Synology requires this.**

```
[ups]
  driver = usbhid-ups
  port = auto
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
sudo systemctl restart nut-server nut-driver
```

---

### 7. (Optional) Open Firewall Port

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
