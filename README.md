# Multiple Isolated Tor Instances for Hidden Services

###### Update: 19.12.2025

Tor is awesome — but let’s be real: sometimes it drives you *absolutely insane*.

So instead of jamming all your hidden services into one massive `torrc` file and hoping for the best, we go the clean route:

🛠️ **One instance per service. Separate configs. Isolated processes.** Each with its own data, control, and behavior.

## Why is this better?

* 🔐 **Better security** – Each instance is sandboxed. No shared state, no accidental leakage.
* 🧠 **Full control** – Individual logs, ports, runtime — total transparency.
* ⚡ **More flexibility** – Restart or debug just one service without touching the others.
* 🧹 **No permission hell** – Avoid weird ownership or `PidFile` issues. Clean, modular.

This guide shows you how to run multiple independent Tor services on the same machine like a pro —

because the one-`torrc`-to-rule-them-all method is fine, until it’s not.

> Build it like a Batmobile: fast, stealthy, unstoppable.

## Table of Contents



## Prerequisites

* Tor installed (`e.g.recommend debian// sudo apt install tor`)
* Apache2 (LAMP) installed and running
* Basic Linux command-line knowledge
* Secure your Server befor, or pay me!

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

# Security Hardening
SafeSocks 1
TestSocks 1
WarnUnsafeSocks 1
DNSPort 0
RelayBandwidthRate 100 KBytes
RelayBandwidthBurst 200 KBytes
NumEntryGuards 8
PathsNeededToBuildCircuits 0.95

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

# Security Hardening
SafeSocks 1
TestSocks 1
WarnUnsafeSocks 1
DNSPort 0
RelayBandwidthRate 100 KBytes
RelayBandwidthBurst 200 KBytes
NumEntryGuards 8
PathsNeededToBuildCircuits 0.95

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

1. **`/etc/systemd/system/tor@hidden_service_1.service`**
2. **`/etc/systemd/system/tor@hidden_service_2.service`**

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

# Security Hardening
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectControlGroups=true
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX
RestrictNamespaces=true
LockPersonality=true
RestrictRealtime=true
RestrictSUIDSGID=true
RemoveIPC=true
SystemCallFilter=@system-service
SystemCallErrorNumber=EPERM

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
Listen 127.0.0.1:9000
Listen 127.0.0.1:9001

```

### 2. Create Web Directories

```bash
sudo mkdir -p /var/www/html/hidden_service_1
sudo mkdir -p /var/www/html/hidden_service_2

```

### 3. Create VirtualHost Files

**`/etc/apache2/sites-available/hidden_service_1.conf`**:

```apache
<VirtualHost 127.0.0.1:9000>
  DocumentRoot /var/www/html/hidden_service_1
  
  # Security Headers
  Header always set X-Frame-Options "DENY"
  Header always set X-Content-Type-Options "nosniff"
  Header always set Referrer-Policy "no-referrer"
  Header always set Permissions-Policy "geolocation=(), microphone=(), camera=()"
  
  # Disable directory listing
  Options -Indexes
  
  <Directory /var/www/html/hidden_service_1>
    Require all granted
    AllowOverride None
  </Directory>
</VirtualHost>

```

**`/etc/apache2/sites-available/hidden_service_2.conf`**:

```apache
<VirtualHost 127.0.0.1:9001>
  DocumentRoot /var/www/html/hidden_service_2
  
  # Security Headers
  Header always set X-Frame-Options "DENY"
  Header always set X-Content-Type-Options "nosniff"
  Header always set Referrer-Policy "no-referrer"
  Header always set Permissions-Policy "geolocation=(), microphone=(), camera=()"
  
  # Disable directory listing
  Options -Indexes
  
  <Directory /var/www/html/hidden_service_2>
    Require all granted
    AllowOverride None
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

## Advanced Security & Scripts

### Firewall Rules (UFW)

```bash
# Better: Restrict to localhost only
sudo ufw deny 9000/tcp
sudo ufw deny 9001/tcp
# Apache only binds to 127.0.0.1

```

### Health Check Script

```bash
#!/bin/bash
# tor-health-check.sh

for service in hidden_service_1 hidden_service_2; do
    if ! systemctl is-active --quiet tor@${service}.service; then
        echo " ${service} is DOWN!"
        # Optional: Auto-restart
        # systemctl restart tor@${service}.service
    fi
done

```

### Log Rotation

Create `/etc/logrotate.d/tor-instances`:

```text
/var/log/tor/instances/*/notices.log {
    daily
    rotate 7
    compress
    delaycompress
    notifempty
    missingok
    postrotate
        systemctl reload tor@*.service > /dev/null 2>&1 || true
    endscript
}

```

### Backup Script for .onion Keys

```bash
#!/bin/bash
# backup-onion-keys.sh

BACKUP_DIR="/root/tor-backups/$(date +%Y%m%d)"
mkdir -p "$BACKUP_DIR"

for service in hidden_service_1 hidden_service_2; do
    cp -r /var/lib/tor/instances/${service}/hidden_service/ \
          "$BACKUP_DIR/${service}/"
    chmod 600 "$BACKUP_DIR/${service}"/*
done

echo "Keys backed up to $BACKUP_DIR"

```

### Monitoring Script

```bash
#!/bin/bash
# monitor-tor-services.sh

for service in hidden_service_1 hidden_service_2; do
    onion=$(cat /var/lib/tor/instances/${service}/hidden_service/hostname)
    port=$(grep HiddenServicePort /etc/tor/instances/${service}/torrc | awk '{print $3}' | cut -d: -f2)
    
    echo "Service: ${service}"
    echo "Onion: ${onion}"
    echo "Port: ${port}"
    echo "Status: $(systemctl is-active tor@${service}.service)"
    echo "---"
done

```

### Fail2Ban for Apache (optional)

Add to `/etc/fail2ban/jail.local`:

```ini
[apache-auth]
enabled = true
port = 9000,9001
logpath = /var/log/apache2/*error.log
maxretry = 3
bantime = 3600

```

### Test Script

```bash
#!/bin/bash
# test-tor-setup.sh

echo "Testing Tor instances..."

for service in hidden_service_1 hidden_service_2; do
    port=$(grep HiddenServicePort /etc/tor/instances/${service}/torrc | awk '{print $3}' | cut -d: -f2)
    
    if curl -s http://127.0.0.1:${port} > /dev/null; then
        echo "${service} responding on ${port}"
    else
        echo "${service} NOT responding on ${port}"
    fi
done

```

---

## Troubleshooting

### Service won't start:

* Check logs: `journalctl -u tor@hidden_service_1 -n 50`
* Verify permissions: `ls -la /var/lib/tor/instances/`
* Test config: `sudo -u debian-tor tor -f /etc/tor/instances/hidden_service_1/torrc --verify-config`

### Can't access .onion:

* Wait 3-5 minutes after first start
* Check Apache: `sudo systemctl status apache2`
* Verify port: `sudo netstat -tulpn | grep :9000`

### Permission errors:

* Re-run step 4 (Set Permissions)
* Check SELinux: `sudo setenforce 0` (temporarily)

---

## Performance Tuning

For better performance on busy services, add these to your `torrc`:

```ini
MaxCircuitDirtiness 600
CircuitBuildTimeout 30
LearnCircuitBuildTimeout 0

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

**Credits:** - Readme.md Powered by Batman’s grind and ChatGPT wizardry. 🦇🔥

