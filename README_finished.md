# Raspberry Pi 5 NAS / PiNAS

A personal Raspberry Pi 5 NAS project built with Docker, Nextcloud, MariaDB, Nginx Proxy Manager, and Tailscale.

This project documents my working Raspberry Pi 5 NAS setup, also known as a **PiNAS**. The goal was to build a private, self-hosted cloud storage system that I can access safely from my own devices without exposing public ports on my home network.

> **Status:** Working personal NAS setup  
> **Main use case:** Private cloud storage, file syncing, and remote access  
> **Remote access method:** Tailscale private VPN access  
> **Public ports required:** None  

---

## Table of Contents

- [Project Overview](#project-overview)
- [Hardware Used](#hardware-used)
- [Architecture](#architecture)
- [Storage Setup](#storage-setup)
- [Docker Setup](#docker-setup)
- [Docker Compose Configuration](#docker-compose-configuration)
- [Start the Containers](#start-the-containers)
- [Nginx Proxy Manager Setup](#nginx-proxy-manager-setup)
- [Tailscale Setup](#tailscale-setup)
- [Nextcloud Trusted Domain Configuration](#nextcloud-trusted-domain-configuration)
- [Testing the Setup](#testing-the-setup)
- [Security Notes](#security-notes)
- [Troubleshooting](#troubleshooting)
- [Future Improvements](#future-improvements)
- [Resources](#resources)

---

## Project Overview

This project turns a Raspberry Pi 5 into a small personal NAS using:

- **Raspberry Pi 5**
- **SATA HAT**
- **Attached SATA storage**
- **RAID0 storage mounted at `/mnt/raid`**
- **Docker**
- **Nextcloud**
- **MariaDB**
- **Nginx Proxy Manager**
- **Tailscale**

The final result is a self-hosted cloud storage system similar to Google Drive, OneDrive, or Dropbox, except the files are stored on my own hardware.

The setup uses **Tailscale** so I can access Nextcloud remotely through a private encrypted network instead of opening ports on my router.

---

## Hardware Used

- Raspberry Pi 5
- SATA HAT for Raspberry Pi 5
- Mini cooler / cooling solution
- SATA storage drives
- MicroSD card or boot drive
- Power supply for Raspberry Pi 5
- Network connection  
  - This setup currently uses **Wi-Fi 5 GHz**

### Hardware Setup Reference

I followed this video for the initial Raspberry Pi 5 SATA HAT hardware setup:

https://www.youtube.com/watch?v=l30sADfDiM8

Make sure to also install any required software or drivers that allow the Raspberry Pi 5 to communicate with the SATA HAT.

### Photo of My PiNAS

<img src="https://github.com/user-attachments/assets/d9412ff1-830b-4c7f-8a11-b7f4d22953e0" style="width:50%; height:auto;" />

---

## Architecture

```text
Client Device
Browser / Nextcloud Mobile App
        |
        | HTTPS through Tailscale
        v
Tailscale MagicDNS / Tailscale Serve
        |
        | HTTP internally
        v
Nginx Proxy Manager
        |
        | HTTP reverse proxy
        v
Nextcloud Docker Container
        |
        v
MariaDB Docker Container
        |
        v
RAID Storage Mounted at /mnt/raid
```

This setup bypasses CGNAT and avoids public port forwarding by using Tailscale. Tailscale provides private encrypted access to the Raspberry Pi. Nginx Proxy Manager then routes the internal HTTP request to the Nextcloud Docker container.

No public DNS records, Cloudflare tunnels, router port forwarding, or public-facing services are required.

---

## Storage Setup

My storage is mounted at:

```bash
/mnt/raid
```

The Nextcloud data directory is stored at:

```bash
/mnt/raid/nextcloud-data
```

Inside the Nextcloud container, that folder is mapped to:

```bash
/var/www/html/data
```

### RAID Choice

For this project, I used **RAID0** to combine storage capacity and use all available drive space.

> **Important:** RAID0 is not a backup. If one drive fails, the entire array can be lost. RAID0 is useful for capacity and performance, but not for redundancy.

If you want redundancy, consider:

- RAID1
- RAID5 / RAIDZ
- External USB backup drive
- Another NAS backup
- Cloud backup for critical files

Create the Nextcloud data directory if it does not already exist:

```bash
sudo mkdir -p /mnt/raid/nextcloud-data
```

Set ownership so the Nextcloud Apache container can write to it:

```bash
sudo chown -R www-data:www-data /mnt/raid/nextcloud-data
```

If needed, set permissions:

```bash
sudo chmod -R 750 /mnt/raid/nextcloud-data
```

Verify the mount:

```bash
df -h
```

You should see `/mnt/raid` listed.

---

## Docker Setup

First, check whether Docker and Docker Compose are already installed:

```bash
docker --version
```

```bash
docker compose version
```

Update the system:

```bash
sudo apt update && sudo apt upgrade -y
```

Install required packages:

```bash
sudo apt-get install ca-certificates curl gnupg lsb-release -y
```

Create the Docker keyrings folder:

```bash
sudo mkdir -p /etc/apt/keyrings
```

Add Docker's official GPG key:

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

Set permissions:

```bash
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

Add the Docker repository:

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Update package lists:

```bash
sudo apt-get update
```

Install Docker:

```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

Verify installation:

```bash
docker --version
```

```bash
docker compose version
```

Optional: allow your user to run Docker commands without typing `sudo`:

```bash
sudo usermod -aG docker $USER
```

Then log out and log back in.

---

## Docker Compose Configuration

Create a project folder:

```bash
mkdir -p ~/nextcloud
```

Move into it:

```bash
cd ~/nextcloud
```

Create the Docker Compose file:

```bash
nano docker-compose.yml
```

Paste the following:

```yaml
version: "3.8"

services:
  db:
    image: mariadb:10.11
    container_name: nextcloud_db
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: "CHANGE_THIS_ROOT_PASSWORD"
      MYSQL_DATABASE: "nextcloud"
      MYSQL_USER: "nextcloud"
      MYSQL_PASSWORD: "CHANGE_THIS_DATABASE_PASSWORD"
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - web

  nextcloud:
    image: nextcloud:32-apache
    container_name: nextcloud
    restart: always
    depends_on:
      - db
    environment:
      MYSQL_HOST: db
      MYSQL_DATABASE: nextcloud
      MYSQL_USER: nextcloud
      MYSQL_PASSWORD: "CHANGE_THIS_DATABASE_PASSWORD"
    volumes:
      - nextcloud_html:/var/www/html
      - /mnt/raid/nextcloud-data:/var/www/html/data
    networks:
      - web

  nginx-proxy-manager:
    image: jc21/nginx-proxy-manager:latest
    container_name: nginx-proxy-manager
    restart: always
    dns:
      - 1.1.1.1
      - 9.9.9.9
    ports:
      - "80:80"
      - "81:81"
      # Port 443 is not needed here because Tailscale provides HTTPS access.
    volumes:
      - npm_data:/data
    networks:
      - web

volumes:
  db_data:
  nextcloud_html:
  npm_data:

networks:
  web:
    driver: bridge
```

> **Important:** Do not commit real passwords to GitHub. Replace the placeholder passwords before running the containers.

---

## Start the Containers

Start everything:

```bash
docker compose up -d
```

Check that the containers are running:

```bash
docker ps
```

You should see containers similar to:

```text
nextcloud_db
nextcloud
nginx-proxy-manager
```

View logs if something does not start correctly:

```bash
docker compose logs
```

Or view logs for a specific container:

```bash
docker logs nextcloud
```

```bash
docker logs nextcloud_db
```

```bash
docker logs nginx-proxy-manager
```

---

## Nginx Proxy Manager Setup

Open Nginx Proxy Manager from another machine on your LAN:

```text
http://<PI_LAN_IP>:81
```

Find the Raspberry Pi's LAN IP with:

```bash
hostname -I
```

Or:

```bash
ip addr
```

The default Nginx Proxy Manager login is:

```text
Email: admin@example.com
Password: changeme
```

After logging in, immediately change the default email and password.

### Add a Proxy Host

In Nginx Proxy Manager, go to:

```text
Hosts -> Proxy Hosts -> Add Proxy Host
```

Use the following settings:

```text
Domain Names: nextcloud.local
Scheme: http
Forward Hostname / IP: nextcloud
Forward Port: 80
```

Enable:

```text
Websockets Support
Block Common Exploits
```

Leave SSL off in Nginx Proxy Manager because Tailscale will handle HTTPS externally.

Save the proxy host and wait for it to show as online.

Later, after Tailscale is fully configured, replace `nextcloud.local` with your Tailscale HTTPS hostname, which will look similar to:

```text
your-pi-name.your-tailnet-name.ts.net
```

---

## Tailscale Setup

Install Tailscale:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Start and authenticate Tailscale:

```bash
sudo tailscale up
```

Tailscale should print a login link. Open the link in your browser and sign in.

Check the Tailscale connection:

```bash
tailscale status
```

Your Raspberry Pi should show as connected.

### Enable MagicDNS

In the Tailscale admin console:

```text
DNS -> MagicDNS -> Enable
```


### Enable HTTPS Certificates

In the Tailscale admin console, enable HTTPS certificates for your tailnet.

Tailscale Serve requires HTTPS certificates before it can provide HTTPS access to a local service.

### Serve Nginx Proxy Manager Through Tailscale

Once Nginx Proxy Manager is listening on local port 80, use Tailscale Serve to expose that internal service privately to devices in your tailnet:

```bash
sudo tailscale serve --bg http://127.0.0.1:80
```

Check the Serve status:

```bash
tailscale serve status
```

You should see a Tailscale HTTPS URL similar to:

```text
https://your-pi-name.your-tailnet-name.ts.net
```

Use that full hostname in Nginx Proxy Manager as the proxy host domain name.

> This is private to your Tailscale network. It is not the same as Tailscale Funnel, which would make a service public.

---

## Nextcloud Trusted Domain Configuration

Nextcloud may block access until the Tailscale hostname is added as a trusted domain.

Enter the Nextcloud container:

```bash
docker exec -it nextcloud bash
```

Run the following commands inside the container.

Add the Tailscale hostname as a trusted domain:

```bash
php occ config:system:set trusted_domains 1 --value="your-pi-name.your-tailnet-name.ts.net"
```

Set the overwrite protocol to HTTPS:

```bash
php occ config:system:set overwriteprotocol --value="https"
```

Set the overwrite host:

```bash
php occ config:system:set overwritehost --value="your-pi-name.your-tailnet-name.ts.net"
```

Exit the container:

```bash
exit
```

Restart Nextcloud:

```bash
docker restart nextcloud
```

Now open:

```text
https://your-pi-name.your-tailnet-name.ts.net
```

---

## Testing the Setup

### Test Docker Containers

```bash
docker ps
```

Expected result:

```text
nextcloud_db
nextcloud
nginx-proxy-manager
```

### Test Local Nginx Proxy Manager

```text
http://<PI_LAN_IP>:81
```

### Test Tailscale Connection

```bash
tailscale status
```

### Test Tailscale Serve

```bash
tailscale serve status
```

### Test Nextcloud Access

From a device signed into the same Tailscale network, open:

```text
https://your-pi-name.your-tailnet-name.ts.net
```

You should see the Nextcloud setup page or login page.

---

## Security Notes

This setup was designed to avoid exposing the NAS directly to the public internet.

Security choices used in this project:

- No router port forwarding
- No public DNS required
- Tailscale private network access
- HTTPS provided through Tailscale
- Nginx Proxy Manager used internally
- MariaDB separated into its own Docker container
- Nextcloud data stored outside the container on mounted RAID storage
- Default Nginx Proxy Manager credentials changed after first login
- Placeholder passwords used in the public README

Recommended additional security improvements:

- Enable 2FA on the Tailscale account
- Enable 2FA on the Nextcloud admin account
- Use strong unique database passwords
- Do not commit secrets to GitHub
- Keep Docker images updated
- Keep Raspberry Pi OS updated
- Use external backups for important files
- Consider disabling or firewalling Nginx Proxy Manager admin port `81` after setup
- Use a UPS if the NAS stores important data

---

## Troubleshooting

### Containers are not running

Check container status:

```bash
docker ps -a
```

View logs:

```bash
docker compose logs
```

Restart:

```bash
docker compose restart
```

---

### Nginx Proxy Manager will not load

Check that port 81 is mapped:

```bash
docker ps
```

Try opening:

```text
http://<PI_LAN_IP>:81
```

Make sure the Raspberry Pi firewall is not blocking port 81 on the LAN.

---

### Nextcloud says untrusted domain

Add your Tailscale hostname to Nextcloud trusted domains:

```bash
docker exec -it nextcloud bash
php occ config:system:set trusted_domains 1 --value="your-pi-name.your-tailnet-name.ts.net"
exit
docker restart nextcloud
```

---

### Nextcloud redirects to HTTP instead of HTTPS

Set overwrite protocol:

```bash
docker exec -it nextcloud bash
php occ config:system:set overwriteprotocol --value="https"
php occ config:system:set overwritehost --value="your-pi-name.your-tailnet-name.ts.net"
exit
docker restart nextcloud
```

---

### Tailscale hostname does not work

Check Tailscale status:

```bash
tailscale status
```

Check Serve status:

```bash
tailscale serve status
```

Make sure MagicDNS and HTTPS certificates are enabled in the Tailscale admin console.

---

### RAID mount is missing after reboot

Check mounts:

```bash
df -h
```

Check block devices:

```bash
lsblk
```

If `/mnt/raid` is not mounted, verify the `/etc/fstab` entry for the RAID array.

---

### Permission issues with Nextcloud data directory

Reset ownership:

```bash
sudo chown -R www-data:www-data /mnt/raid/nextcloud-data
```

Restart Nextcloud:

```bash
docker restart nextcloud
```

---

## Future Improvements

Possible upgrades for this project:

- Move from Wi-Fi to Ethernet for better NAS performance
- Add automated external backups
- Switch from RAID0 to RAID1 or another redundant storage setup
- Add monitoring with Uptime Kuma, Grafana, or Netdata
- Add a UPS for safer shutdowns during power loss
- Use Docker `.env` files or Docker secrets instead of hardcoded passwords
- Add firewall rules to restrict access to Nginx Proxy Manager admin port
- Add screenshots of the final Nextcloud dashboard
- Add benchmark results for read/write speeds
- Document the RAID creation process in more detail

---

## Resources

- Raspberry Pi 5 SATA HAT setup video: https://www.youtube.com/watch?v=l30sADfDiM8
- Docker documentation: https://docs.docker.com/
- Nextcloud documentation: https://docs.nextcloud.com/
- MariaDB Docker image: https://hub.docker.com/_/mariadb
- Nextcloud Docker image: https://hub.docker.com/_/nextcloud
- Nginx Proxy Manager: https://nginxproxymanager.com/
- Tailscale documentation: https://tailscale.com/docs/
- Tailscale Serve documentation: https://tailscale.com/docs/features/tailscale-serve

---

## Disclaimer

This project is for personal homelab and learning purposes. Anyone following this setup should use strong passwords, keep backups, and understand that RAID0 does not protect against drive failure.
