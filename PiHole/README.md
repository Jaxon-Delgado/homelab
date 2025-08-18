# Installing PiHole as a DNS server and network-wide Ad-blocker

## Overview

This project was a great learning project into DNS services and creating my own local domain. It gave me a peek into networking and learning exactly how routing through DNS services works. Having your own DNS server is also highly beneficial for home lab environments using containerization and virtualization because, as we all know, remembering all of those IP addresses can be a pain! Using PiHole as a DNS service, you can setup local DNS entries so instead of typing in those pesky 192.168.... , you instead simply use a subdomain along with whatever top-level domain you choose!

---

## Objectives

1. Spin up a Proxmox VM to host **[PiHole](https://pi-hole.net/)** that has adequate resources.
    _In our case, we used the [Proxmox VE Helper-Scripts](https://community-scripts.github.io/ProxmoxVE/) to automatically spin up an LXC container with the advanced options listed in Step 1._
2. Define the PiHole LXC container as the primary DNS address in the router settings.
3. Setup Local DNS entries in **PiHole** that point towards an Nginx Reverse Proxy.
4. Use Nginx Reverse Proxy to point FQDNs to each service and port.
5. Demonstrate functionality by navigating to the FQDNs of each service.

---

## Hardware and Software Requirements

### Hardware

- Proxmox installation
- At least 2GB RAM
- At least 2 CPU cores

### Software

- LXC Container with ~2GB RAM and ~2 CPU cores dedicated to it
- **PiHole** installation script
- **Nginx Reverse Proxy Manager** (Increased security)
- Services on your network to point DNS entries to

---

## Step 1: Setting up the LXC Container

In our case, we used the [Proxmox VE Helper-Scripts](https://community-scripts.github.io/ProxmoxVE/).

_Prerequisites:_

- _1. Access to your Proxmox's shell (whether through SSH or the management GUI)_
- _2. Basic knowledge of Linux commands_

### Steps

- Navigate to the Helper Scripts website into the "Adblock & DNS" Section.
- Copy the installation script:

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/pihole.sh)"
```

- Paste into your Proxmox shell and execute.
- Use option 3: Advanced Settings for extra customization (Use Default Settings if you don't want to customize anything)
- Select Unprivileged
- Type in a **secure** password (THIS IS FOR ROOT ACCESS! MAKE IT SECURE!)
- Set your Container ID and hostname
- Set desired Disk Size in GB (Default of 2 should be fine in our case)
- Allocate CPU cores (2) and RAM (2048 MB)
- Use all other defaults until you get to IP.
- Assign PiHole a static IP address using CIDR notation (192.168.1.x/24 OR 10.0.0.x/24 OR whatever you have set.)
- Enter your default gateway (Router's IP address)
- Enter any SSH keys you'd like, otherwise skip
- Do **NOT** enable Root SSH access (Default No)
- Use all defaults and create PiHole LXC

## Step 2: Configure PiHole

1. SSH into the PiHole LXC (or use Proxmox's built-in shell) and use:

```pihole setpassword```

to set the admin users password.
2. In a web browser, navigate to the static IP address  you set in the LXC creation as:

```<your_ip>/admin/```

this will bring up a login screen
3. Login using the password you just set.
Welcome to the PiHole GUI, play around a little and get familiar with the interface.

---

## Step 3: Set PiHole as primary DNS and add local DNS entries

***Note, this step is dependent on your ISP. There are many tutorials available online to learn how to change your router's DNS server addresses. In our case, we will be doing it through Verizon FIOS**_

### Changing your router's DNS server

1. Login to your routers webpage (Enter the Default Gateway address into a web browser)
2. Navigate to Advanced -> Network Settings -> Network Connections -> Broadband Connection (Ethernet) and click 'Edit'
3. The bottom of the page hosts the button `Settings`. Click that and find 'IPv4 DNS Address 1:'
4. Change that address to the `static IP address` you set for the PiHole LXC Container
5. Set the 'IPv4 DNS Address 2:' as either `8.8.8.8` (Google's DNS address) or your ISP's DNS server address.
6. Hit apply and save all changes!

### Adding local DNS entries in PiHole

1. Navigate to the PiHole GUI
2. On the left side, select `Settings` then `Local DNS Records`
3. In the Local DNS Entries Section, add in a `FQDN` on the left, and then your `Nginx Reverse Proxy Manager IP Address` on the right.
4. In our case, we added `proxmox.homenet.lan` on the left, and `10.0.0.25` on the right.
5. We then hit the Green Plus sign and added the local DNS entry!

---

## Step 4: Configuring Nginx to forward requests to the correct IP and port

1. Navigate to the Nginx Reverse Proxy Manager GUI.
2. Click on Proxy Hosts to open the proxy screen.
3. Click `Add Proxy Host` and fill in the following information:

- **Domain Names** (The internal domain you have, proxmox.homenet.lan in our case)
- **Scheme** (https)
- **Forward Hostname / IP** (The IP address of your service, 10.0.0.25 in our case)
- **Forward Port** (The port that your service is running on)
- **Check Block Common Exploits and Websockets Support**
- **Navigate to the SSL tab**
- **Check the SSL certificate you have setup for the local domain**
- **Click Save**

---

## Step 5: Check functionality

Now the last thing we need to do is navigate to `proxmox.homenet.lan` in our browser and we should be able to login to our Proxmox instance!
