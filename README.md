# RaspberryPi5-NAS
As of December 2025 this is a working RaspberryPi5 setup as a personal NAS. Otherwise known as a PiNAS. I will be giving documentation on everything I did to make this possible and including YouTube links for hardware setup. If you're looking to make a free-secure-self-hosted-cloud(excluding hardware) you can access from anywhere safely then this is the project for you!<br/>
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Hardware(+SATA hat config):<br/>
Follow this video tutorial on how to initially setup the Pi5 with the SATA hat addition and mini cooler: https://www.youtube.com/watch?v=l30sADfDiM8 <br/>
Also, make sure you follow the steps for installing the software that allows the Pi5 to comunicate with the SATA hat. For RAID configuration I chose RAID0 to utilize all storage space. If you'd rather have backups then something like a RAID1 configuraiton is for you. <br/>
A photo of my complete PiNAS utilizing WIFI 5 5 GHz: <br/>
<img src="https://github.com/user-attachments/assets/d9412ff1-830b-4c7f-8a11-b7f4d22953e0" style="width:50%; height:auto;" />
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Architecture:<br/>
Client (Browser / Mobile App)<br/>
        │<br/>
        │ HTTPS (WireGuard via Tailscale)<br/>
        ▼<br/>
Tailscale Magic DNS<br/>
        │<br/>
        ▼<br/>
Nginx Proxy Manager (HTTP, internal only)<br/>
        │<br/>
        ▼<br/>
Nextcloud (Docker)<br/>
        │<br/>
        ▼<br/>
MariaDB (Docker)<br/>
<br/>
This setup bypasses CGNAT by using Tailscale to provide private, encrypted HTTPS access. Tailscale terminates TLS at the network edge and forwards traffic over HTTP to Nginx Proxy Manager, which then reverse-proxies requests to Nextcloud internally. No public ports or public DNS are required.<br/>
