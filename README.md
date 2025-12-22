# RaspberryPi5-NAS
As of December 2025 this is a working RaspberryPi5 setup as a personal NAS. Otherwise known as a PiNAS. I will be giving documentation on everything I did to make this possible and including YouTube links for hardware setup. If you're looking to make a free-secure-self-hosted-cloud(excluding hardware) you can access from anywhere safely then this is the project for you!<br/>
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Hardware(+SATA hat config):<br/>
Follow this video tutorial on how to initially setup the Pi5 with the SATA hat addition and mini cooler: https://www.youtube.com/watch?v=l30sADfDiM8 <br/>
Also, make sure you follow the steps for installing the software that allows the Pi5 to comunicate with the SATA hat. For RAID configuration I chose RAID0 to utilize all storage space. If you'd rather have backups then something like a RAID1 configuraiton is for you. <br/>
A photo of my complete PiNAS utilizing WIFI 5 5 GHz: <br/>
<img src="https://github.com/user-attachments/assets/d9412ff1-830b-4c7f-8a11-b7f4d22953e0" style="width:50%; height:auto;" />
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
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
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Step-by-step:
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Docker:<br/>
Check to make sure latest docker and docker compose versions are installed with<br/>
"docker --version"<br/>
and <br/>
"docker compose version"<br/>
<br/>
If not the latest version go ahead and update everything with:<br/>
"sudo apt update && sudo apt upgrade -y"<br/>
<br/>
If not installed then install with(these commands one by one):<br/>
"sudo apt-get install ca-certificates curl gnupg lsb-release -y"<br/>
"sudo mkdir -p /etc/apt/keyrings"<br/>
"curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg"<br/>
"sudo chmod a+r /etc/apt/keyrings/docker.gpg"<br/>
"echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null"<br/>
"sudo apt-get update"<br/>
"sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y"<br/>
<br/>
Then go ahead and reverify instalation and version.<br/>
<br/>
Next step is to create the project folder that'll contain the docker compose yaml file:<br/>
"mkdir -p ~/nextcloud"<br/>
"cd ~/nextcloud"<br/>
<br/>
Next while in the nextcloud folder we will create the yaml:<br/>
"nano docker-compose.yml"<br/>
<br/>
As of December 2025 this is exactly how your yaml should look:<br/>
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
version: "3.8"<br/>
<br/>
services:<br/>
  db:<br/>
    image: mariadb:10.11<br/>
    container_name: nextcloud_db<br/>
    restart: always<br/>
    environment:  # using mysql is optional but suggested for a security layer<br/>
      MYSQL_ROOT_PASSWORD: "password"  # you'll be putting your own password here<br/>
      MYSQL_DATABASE: "nextcloud"  <br/>
      MYSQL_USER: "nextcloud"  <br/>
      MYSQL_PASSWORD: "password"  # you'll be putting your own password here<br/>
    volumes:<br/>
      - db_data:/var/lib/mysql<br/>
    networks:<br/>
      - web<br/>
<br/>
  nextcloud:<br/>
    image: nextcloud:32-apache<br/>
    container_name: nextcloud<br/>
    restart: always<br/>
    depends_on:<br/>
      - db<br/>
    environment:<br/>
      MYSQL_HOST: db<br/>
      MYSQL_DATABASE: nextcloud<br/>
      MYSQL_USER: nextcloud<br/>
      MYSQL_PASSWORD: password  # you'll be putting your own password here<br/>
    volumes:<br/>
      - nextcloud_html:/var/www/html<br/>
      - /mnt/raid/nextcloud-data:/var/www/html/data<br/>
    networks:<br/>
      - web<br/>

  nginx-proxy-manager:<br/>
    image: jc21/nginx-proxy-manager:latest<br/>
    container_name: nginx-proxy-manager<br/>
    restart: always<br/>
    dns:<br/>
      - 1.1.1.1<br/>
      - 9.9.9.9<br/>
    ports:<br/>
      - "80:80"     # internal http entry (Tailscale will hit this)<br/>
      - "81:81"     # NPM admin UI<br/>
      # NOTE: we do NOT need 443 because Tailscale provides HTTPS externally<br/>
    volumes:<br/>
      - npm_data:/data<br/>
    networks:<br/>
      - web<br/>
<br/>
volumes:<br/>
  db_data:<br/>
  nextcloud_data:<br/>
  npm_data:<br/>
<br/>
networks:<br/>
  web:<br/>
    driver: bridge<br/>
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Now start the containers:<br/>
"docker compose up -d"<br/>
"docker ps"<br/>
You should see nextcloud_db, nextcloud, and nginx-proxy-manager<br/>
That's all to do with docker for now!<br/>
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
