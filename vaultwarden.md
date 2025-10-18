

**Step 1: Install Docker Engine**

Run these commands:

`# Add Docker's official GPG key`  
`sudo apt-get update`  
`sudo apt-get install -y ca-certificates curl`  
`sudo install -m 0755 -d /etc/apt/keyrings`  
`sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc`  
`sudo chmod a+r /etc/apt/keyrings/docker.asc`

`# Add the repository to Apt sources`  
`echo \`  
  `"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \`  
  `https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \`  
  `sudo tee /etc/apt/sources.list.d/docker.list > /dev/null`

`sudo apt-get update`

`# Install Docker and Compose`  
`sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin`

## **Step 2: Set Up Nginx Proxy Manager**

This will manage your reverse proxy, SSL, and subdomains easily.

`mkdir -p ~/docker/npm`  
`cd ~/docker/npm`  
`nano docker-compose.yml`

Paste:

`services:`  
  `nginx_proxy_manager:`  
    `image: jc21/nginx-proxy-manager:latest`  
    `container_name: nginx_proxy_manager`  
    `restart: unless-stopped`  
    `ports:`  
      `- "80:80"`  
      `- "443:443"`  
      `- "81:81"`  
    `volumes:`  
      `- ./data/npm_data:/data`  
      `- ./data/npm_letsencrypt:/etc/letsencrypt`  
      `- ./data/npm_logs:/var/log/nginx`

  `goaccess:`  
    `image: justsky/goaccess-for-nginxproxymanager:latest`  
    `container_name: goaccess`  
    `restart: unless-stopped`  
    `environment:`  
      `- TZ=Etc/UTC`  
    `ports:`  
      `- "7880:7880"`  
    `volumes:`  
      `- ./data/npm_data/logs:/opt/log`

Start:

`sudo docker compose up -d`

Then open your browser:

`http://localhost:81`

Default login:

* **Email:** `admin@example.com`

* **Password:** `changeme`

## **Step 3: Configure DNS (Tailscale \+ Cloudflare)**

If using **Tailscale** (private access only):

1. Get your Tailscale IPv4 (e.g. `100.x.x.x`).

2. In your DNS provider (like Cloudflare), create a record:

   * **Type:** A

   * **Name:** `*.tail`

   * **IPv4 address:** `100.x.x.x`

   * **Proxy status:** DNS only (gray cloud)

3. Save.

If using **Cloudflare Tunnel** (public HTTPS):

You can skip this DNS record and set up the tunnel in Step 9 instead.

## **Step 4: Deploy Vaultwarden**

`cd ~/docker`  
`mkdir vault`  
`cd vault`  
`nano docker-compose.yml`

Paste:

`services:`  
  `vaultwarden:`  
    `image: vaultwarden/server:latest`  
    `container_name: vaultwarden`  
    `restart: unless-stopped`  
    `environment:`  
      `DOMAIN: "http://vault.tail.yourdomain.tld"`  
      `WEBSOCKET_ENABLED: "true"`  
    `volumes:`  
      `- ./vw-data:/data`  
    `ports:`  
      `- "1776:80"`

Start it:

`sudo docker compose up -d`

## **Step 5: Configure Proxy Host (Nginx Proxy Manager)**

1. Open NPM → **Hosts → Proxy Hosts → Add Proxy Host**

2. Fill in:

   * **Domain Name:** `vault.tail.yourdomain.tld`

   * **Scheme:** `http`

   * **Forward Hostname/IP:** `127.0.0.1`

   * **Forward Port:** `1776`

   * Enable ✅ **Websocket Support**

   * Enable ✅ **Block Common Exploits**

   * **Access List:** Publicly Accessible

3. Under **SSL** tab:

   * Certificate: **None**

   * Force SSL: OFF

   * HSTS: OFF

4. Save.

Test it:

`http://vault.tail.yourdomain.tld`

## **Step 6: Create a Vaultwarden Account**

1. Open your Vaultwarden URL.

2. Create an account and log in.

3. Go to **Settings → Security → Two-step Login** and set up an authenticator app.

4. Save your recovery codes in a safe place.

To connect the Bitwarden app or browser extension:

* Open **Settings → Server URL**

Enter:

 `http://vault.tail.yourdomain.tld`

*   
* Make sure your device is connected to **Tailscale**.

## **Step 7: Add Admin Page**

Generate a secure admin token:

`sudo docker exec -it vaultwarden /vaultwarden hash`

Copy the output line:

`ADMIN_TOKEN='your_hashed_token_here'`

Then:

`cd ~/docker/vault`  
`nano .env`

Paste:

`ADMIN_TOKEN=your_hashed_token_here`

Edit `docker-compose.yml` → under `environment:` add:

 `ADMIN_TOKEN: "${ADMIN_TOKEN}"`

Restart Vaultwarden:

`sudo docker compose down`  
`sudo docker compose up -d`

Now you can access:

`http://vault.tail.yourdomain.tld/admin`  
