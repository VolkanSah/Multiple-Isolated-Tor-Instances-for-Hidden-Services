# Multiple Tor Instances for Hidden Services/Tunnels
###### Basics

###### Image K.I generated

Run multiple Tor instances with different configuration files (torrc) to host different hidden services/tunnels on the same machine. This setup is particularly useful when working with tools like SoCat for offensive security purposes.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Setup Instructions](#setup-instructions)
  - [1. Install Tor Service](#1-install-tor-service)
  - [2. Create Configuration Directories](#2-create-configuration-directories)
  - [3. Create Configuration Files](#3-create-configuration-files)
  - [4. Set Up Directories and Permissions](#4-set-up-directories-and-permissions)
  - [5. Create Systemd Service Files](#5-create-systemd-service-files)
  - [6. Reload Systemd and Start Services](#6-reload-systemd-and-start-services)
  - [7. Check Service Status](#7-check-service-status)
  - [8. Retrieve .onion Addresses](#8-retrieve-onion-addresses)
- [Ethical Guidelines](#ethical-guidelines)
- [License](#license)

## Prerequisites

Make sure you have the Tor service installed on your system.

## Setup Instructions

### 1. Install Tor Service

Ensure Tor is installed on your system:
```
sudo apt-get install tor
```

### 2. Create Configuration Directories

Create a new directory for each hidden service configuration:
```
sudo mkdir -p /etc/tor/instances/hidden_service_1
sudo mkdir -p /etc/tor/instances/hidden_service_2
```
Replace `hidden_service_1` and `hidden_service_2` with the desired names for your hidden services.

Create Torfiles:

```
sudo mkdir -p /var/lib/tor/hidden_service_1
sudo mkdir -p /var/lib/tor/hidden_service_2
```

### 3. Create Configuration Files

Create new configuration files for each hidden service:
```
sudo nano /etc/tor/instances/hidden_service_1/torrc
sudo nano /etc/tor/instances/hidden_service_2/torrc
```
Add the following configuration to each torrc file, adjusting the `HiddenServiceDir` and `HiddenServicePort` values as needed:
```
RunAsDaemon 1
DataDirectory /var/lib/tor/instances/hidden_service_1
PidFile /var/run/tor/instances/hidden_service_1.pid
SocksPort 0 
ExitRelay 0  
HiddenServiceDir /var/lib/tor/hidden_service_1/
HiddenServicePort 80 127.0.0.1:9000
```

```
RunAsDaemon 1
DataDirectory /var/lib/tor/instances/hidden_service_2
PidFile /var/run/tor/instances/hidden_service_2.pid
SocksPort 0 
ExitRelay 0  
HiddenServiceDir /var/lib/tor/hidden_service_2/
HiddenServicePort 80 127.0.0.1:9001
```
Replace `hidden_service_1` with the appropriate hidden service name, and adjust the `HiddenServicePort` value to point to the correct local address and port for the associated web service (e.g., Nginx reverse proxy listening on a different port for each hidden service).

Save the changes and exit the editor.

### 4. Set Up Directories and Permissions

Create the required directories and set the correct ownership for the data directories:
```
sudo mkdir -p /var/lib/tor/instances/hidden_service_1
sudo mkdir -p /var/lib/tor/instances/hidden_service_2
sudo chown -R debian-tor:debian-tor /var/lib/tor/instances
```

### 5. Create Systemd Service Files

Create a new systemd service file for each Tor instance:
```
sudo nano /etc/systemd/system/tor@hidden_service_1.service
sudo nano /etc/systemd/system/tor@hidden_service_2.service
```
Add the following content to each systemd service file, adjusting the `hidden_service_1` and `hidden_service_2` values as needed:
```
[Unit]
Description=Tor Hidden Service %I
After=network.target

[Service]
User=debian-tor
Type=simple
ExecStart=/usr/sbin/tor -f /etc/tor/instances/%I/torrc
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
Save the changes and exit the editor.

### 6. Reload Systemd and Start Services

Reload the systemd configuration:
```
sudo systemctl daemon-reload
```
Enable and start the new Tor instances:
```
sudo systemctl enable tor@hidden_service_1.service
sudo systemctl start tor@hidden_service_1.service
sudo systemctl enable tor@hidden_service_2.service
sudo systemctl start tor@hidden_service_2.service
```

### 7. Check Service Status

Check the status of the new Tor instances to ensure they are running correctly:
```
sudo systemctl status tor@hidden_service_1.service
sudo systemctl status tor@hidden_service_2.service
```

### 8. Retrieve .onion Addresses

Retrieve the .onion addresses for your hidden services:
```
sudo cat /var/lib/tor/hidden_service_1/hostname
sudo cat /var/lib/tor/hidden_service_2/hostname
```

Note the .onion addresses for each hidden service, as you will use them to access the respective services.

Now you have multiple Tor instances running with different torrc files, each hosting a separate hidden service/tunnel.

## Ethical Guidelines

This information is intended for educational purposes and to support ethical security testing. It is crucial to use these instructions responsibly. Unauthorized use of this knowledge for malicious purposes is strictly prohibited.

- **No Unauthorized Use**: Do not use these instructions to access, disrupt, or compromise any systems or networks without explicit permission from the owner.
- **Ethical Responsibility**: Use these instructions to improve security, understand network anonymity, and for lawful research and educational purposes only.
- **No Support for Malicious Activities**: Any requests for assistance with illegal activities will be ignored.

## Your Support
If you find this project useful and want to support it, there are several ways to do so:

- If you find the white paper helpful, please ⭐ it on GitHub. This helps make the project more visible and reach more people.
- Become a Follower: If you're interested in updates and future improvements, please follow my GitHub account. This way you'll always stay up-to-date.
- Learn more about my work: I invite you to check out all of my work on GitHub and visit my developer site https://volkansah.github.io. Here you will find detailed information about me and my projects.
- Share the project: If you know someone who could benefit from this project, please share it. The more people who can use it, the better.
**If you appreciate my work and would like to support it, please visit my [GitHub Sponsor page](https://github.com/sponsors/volkansah). Any type of support is warmly welcomed and helps me to further improve and expand my work.**

Thank you for your support! ❤️ S. Volkan Kücükbudak

## License

This project is licensed under the MIT - see the [LICENSE](LICENSE) file for details.
