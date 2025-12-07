# Multiple Isolated Tor Instances for Hidden Services
###### Update: 06.12.2025
![Tor Logo](https://upload.wikimedia.org/wikipedia/commons/1/15/Tor-logo-2011-flat.svg)

Tor is awesome — but let’s be real: sometimes it drives you *absolutely insane*.  
So instead of jamming all your hidden services into one massive `torrc` file and hoping for the best, we go the clean route:

🛠️ **One instance per service. Separate configs. Isolated processes.**  
Each with its own data, control, and behavior.

## Why is this better?

- 🔐 **Better security** – Each instance is sandboxed. No shared state, no accidental leakage.
- 🧠 **Full control** – Individual logs, ports, runtime — total transparency.
- ⚡ **More flexibility** – Restart or debug just one service without touching the others.
- 🧹 **No permission hell** – Avoid weird ownership or `PidFile` issues. Clean, modular.

This guide shows you how to run multiple independent Tor services on the same machine like a pro —  
because the one-`torrc`-to-rule-them-all method is fine, until it’s not.

> Build it like a Batmobile: fast, stealthy, unstoppable.


## Table of Contents

1. [Prerequisites](#prerequisites)  
2. [Setup Instructions](#setup-instructions)  
   1. [Install Tor](#1-install-tor)  
   2. [Create Config & Data Directories](#2-create-config--data-directories)  
   3. [Write Tor Config Files](#3-write-tor-config-files)  
   4. [Set Permissions](#4-set-permissions)  
   5. [Create Systemd Service Files](#5-create-systemd-service-files)  
   6. [Enable and Start Services](#6-enable-and-start-services)  
   7. [Verify Services](#7-verify-services)  
   8. [Get .onion Addresses](#8-get-onion-addresses)  
3. [Configure Apache VirtualHosts](#9-configure-apache-virtualhosts)  
4. [Cleanup & Reset](#cleanup--reset)  
5. [Ethical Use](#ethical-use)  
6. [Support](#support)  
7. [License](#license)

---

## Prerequisites

- Tor installed (`e.g. sudo apt install tor`)
- Apache2 (LAMP) installed and running
- Basic Linux command-line knowledge
- Secure your Server befor, or pay me!


## Setup Instructions

##### 1. Install Tor! On your system



### 2. Create Config & Data Directories

```bash
sudo mkdir -p /etc/tor/instances/hidden_service_1
sudo mkdir -p /etc/tor/instances/hidden_service_2
sudo mkdir -p /var/lib/tor/instances/hidden_service_1/hidden_service
sudo mkdir -p /var/lib/tor/instances/hidden_service_2/hidden_service
```

### 3. Write Tor Config Files

**`/etc/tor/instances/hidden_service_1/torrc`**:

```ini
RunAsDaemon 0
DataDirectory /var/lib/tor/instances/hidden_service_1
PidFile /run/tor/instances/hidden_service_1/hidden_service_1.pid
SocksPort 0
ExitRelay 0
ControlPort 0
HiddenServiceDir /var/lib/tor/instances/hidden_service_1/hidden_service/
HiddenServicePort 80 127.0.0.1:9000
Log notice syslog # for dev!
# Log notice file /dev/null  # for production
```

**`/etc/tor/instances/hidden_service_2/torrc`**:

```ini
RunAsDaemon 0
DataDirectory /var/lib/tor/instances/hidden_service_2
PidFile /run/tor/instances/hidden_service_2/hidden_service_2.pid
SocksPort 0
ExitRelay 0
ControlPort 0
HiddenServiceDir /var/lib/tor/instances/hidden_service_2/hidden_service/
HiddenServicePort 80 127.0.0.1:9001
Log notice syslog # for dev!
# Log notice file /dev/null   # for production
```

### 4. Set Permissions

```bash
sudo chown -R debian-tor:debian-tor /etc/tor/instances
sudo chown -R debian-tor:debian-tor /var/lib/tor/instances
sudo chmod 700 /var/lib/tor/instances/hidden_service_1/hidden_service
sudo chmod 700 /var/lib/tor/instances/hidden_service_2/hidden_service
sudo mkdir -p /run/tor/instances/hidden_service_1
sudo mkdir -p /run/tor/instances/hidden_service_2
sudo chown -R debian-tor:debian-tor /run/tor/instances
```

### 5. Create Systemd Service Files

**`/etc/systemd/system/tor@hidden_service_1.service`**
**`/etc/systemd/system/tor@hidden_service_2.service`**

```ini
[Unit]
Description=Tor Hidden Service %i
After=network.target

[Service]
User=debian-tor
Type=simple
ExecStart=/usr/sbin/tor -f /etc/tor/instances/%i/torrc
Restart=on-failure
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=yes
ReadWritePaths=/var/lib/tor/instances/%i /run/tor/instances/%i
RuntimeDirectory=tor/instances/%i
RuntimeDirectoryMode=0750

[Install]
WantedBy=multi-user.target
```

### 6. Enable and Start Services

```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable --now tor@hidden_service_1.service
sudo systemctl enable --now tor@hidden_service_2.service
```

### 7. Verify Services

```bash
sudo systemctl status tor@hidden_service_1.service
sudo systemctl status tor@hidden_service_2.service
```

Or live logs:

```bash
journalctl -u tor@hidden_service_1.service -f
journalctl -u tor@hidden_service_2.service -f
```

### 8. Get .onion Addresses

```bash
sudo cat /var/lib/tor/instances/hidden_service_1/hidden_service/hostname
sudo cat /var/lib/tor/instances/hidden_service_2/hidden_service/hostname
```



## 9. Configure Apache VirtualHosts

### 1. Open Ports

Edit `/etc/apache2/ports.conf`:

```apache
Listen 9000
Listen 9001
```

### 2. Create Web Directories

```bash
sudo mkdir -p /var/www/html/hidden_service_1
sudo mkdir -p /var/www/html/hidden_service_2
```

### 3. Create VirtualHost Files

**`/etc/apache2/sites-available/hidden_service_1.conf`**:

```apache
<VirtualHost *:9000>
  DocumentRoot /var/www/html/hidden_service_1
  <Directory /var/www/html/hidden_service_1>
    Require all granted
  </Directory>
</VirtualHost>
```

**`/etc/apache2/sites-available/hidden_service_2.conf`**:

```apache
<VirtualHost *:9001>
  DocumentRoot /var/www/html/hidden_service_2
  <Directory /var/www/html/hidden_service_2>
    Require all granted
  </Directory>
</VirtualHost>
```

### 4. Enable Sites & Restart Apache

```bash
sudo a2ensite hidden_service_1.conf
sudo a2ensite hidden_service_2.conf
sudo systemctl restart apache2
```
or (optional)

```bash
sudo a2ensite hidden_service_1 hidden_service_2
sudo systemctl restart apache2
```

### 5. Set Permissions

```bash
sudo chown -R www-data:debian-tor /var/www/html/hidden_service_*
sudo chmod -R 750 /var/www/html/hidden_service_*
```

### 6. Restart Tor

```bash
sudo systemctl restart tor@hidden_service_1
sudo systemctl restart tor@hidden_service_2
```

---

## Cleanup & Reset

```bash
sudo systemctl stop tor@hidden_service_1.service tor@hidden_service_2.service
sudo systemctl disable tor@hidden_service_1.service tor@hidden_service_2.service

sudo rm -rf /etc/tor/instances/hidden_service_1
sudo rm -rf /etc/tor/instances/hidden_service_2
sudo rm -rf /var/lib/tor/instances/hidden_service_1
sudo rm -rf /var/lib/tor/instances/hidden_service_2
sudo rm -rf /run/tor/instances/hidden_service_1
sudo rm -rf /run/tor/instances/hidden_service_2

sudo rm /etc/systemd/system/tor@hidden_service_1.service
sudo rm /etc/systemd/system/tor@hidden_service_2.service

sudo systemctl daemon-reload

# Optional full uninstall
sudo apt purge --auto-remove tor
sudo rm -rf /etc/tor /var/lib/tor /var/log/tor
```

---

## Ethical Use

Use responsibly and legally.
This project is for research, learning, privacy protection, and legal penetration testing.

---

## Support

If this helped you:

* ⭐ the repo
* Share it
* Visit my Profile [Volkan Sah](https://github.com/volkansah) and follow me 😃
* [Support via GitHub Sponsors](https://github.com/sponsors/volkansah)

---

## License

MIT License — see LICENSE file.

---

**Credits:** 
- Readme.md Powered by Batman’s grind and ChatGPT wizardry. 🦇🔥



