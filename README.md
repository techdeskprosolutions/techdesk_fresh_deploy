# techdesk_fresh_deploy
**BETA** Fresh copy of TechDesk Panel.

# homelab-core

**Secure, lightweight Docker Compose foundation** for your self-hosted server.

Includes:
- **Nginx Proxy Manager (NPM)** — Reverse proxy + automatic Let's Encrypt SSL
- **MariaDB** + **phpMyAdmin** — Database server with web management UI
- **wg-easy** — Simple WireGuard VPN server with web UI

**Design goals**: Secure by default (network isolation + NPM Access Lists + VPN), lightweight, and easy to extend with new projects later via the shared `homelab-proxy` network.

---

## Prerequisites

Before starting:

1. A Linux server with **Docker** and **Docker Compose** installed (or Portainer / another Docker management UI).
2. **Router port forwarding**:
   - TCP 80 and 443 → your server (for NPM / web traffic)
   - UDP 51820 → your server (for WireGuard VPN)
3. Server firewall (ufw, firewalld, etc.) allows the above ports.
4. (Strongly recommended) A domain name with A record(s) pointing to your server's public IP.
5. SSH access to the server.

---

## Step-by-Step Setup (After Deploying via Docker Manager)

These steps assume you have deployed (or are about to deploy) using the raw `docker-compose.yml` URL in your Docker management tool (Portainer Stack, etc.).

### 1. Prepare the Server Directory (Recommended)

Even if deploying via raw URL, it's best to work in a dedicated folder:

```bash
mkdir -p ~/homelab-core/data/{npm,mariadb,wg-easy}
cd ~/homelab-core
```

> **Note for raw URL / Portainer users**: Create the folder and `.env` file on the server first. Then deploy the stack from the GitHub raw URL of `docker-compose.yml`. Relative bind mounts will work best when the compose context has the `data/` folders.

### 2. Get the Files

**Option A – Clone (recommended)**

```bash
git clone https://github.com/YOUR_USERNAME/homelab-core.git
cd homelab-core
```

**Option B – Manual / Raw URL users**

Download these two files into your `~/homelab-core` folder:
- `docker-compose.yml` (raw from GitHub)
- `.env.example`

### 3. Configure Environment Variables (CRITICAL)

```bash
cp .env.example .env
nano .env          # or use your favorite editor
```

**Edit `.env` and change EVERYTHING marked with comments in ALL CAPS.**

Key fields you **MUST** update:

- `DB_ROOT_PASSWORD` — Strong unique password for MariaDB root
- `WG_HOST` — Your public IP **or** domain name
- `WG_PASSWORD` — Strong password for the wg-easy web UI

**Generate strong passwords** (run on your server):

```bash
openssl rand -base64 32
```

Save the file.

### 4. Deploy the Stack

**Using Docker Compose CLI (recommended for first run):**

```bash
docker compose up -d
```

**Using Docker Manager / Portainer (raw URL method):**

1. In Portainer (or your manager), go to **Stacks** → **Add stack**.
2. Choose **"Repository"** or **"URL"** and paste the **raw URL** of `docker-compose.yml` from GitHub.
3. Provide the environment variables from your `.env` file in the UI (or ensure the `.env` file exists in the deployment context).
4. Deploy the stack.

Wait 30–60 seconds for containers to start.

Verify everything is running:

```bash
docker compose ps
# or in Portainer: check the stack containers
```

### 5. Initial Access – Nginx Proxy Manager

1. Open your browser and go to:  
   **`http://YOUR-SERVER-PUBLIC-IP:81`**

2. Default login:
   - Email: `admin@example.com`
   - Password: `changeme`

3. **Immediately change** the admin email and password (Settings → Users or the prompt).

4. (Optional but recommended) Create a proxy host for NPM itself later so you can access the admin UI over HTTPS + your Access List.

### 6. Create an NPM Access List (for Admin Interfaces)

This is the key security step.

1. In NPM → **Access Lists** → **Add Access List**
2. Name it something like `Admins - Home + VPN`
3. Add your **home public IP** (use `/32` for single IP, e.g. `203.0.113.50/32`)
4. Add the **WireGuard client subnet** (default is usually `10.8.0.0/24` — you can confirm later in wg-easy)
5. Save.

You will apply this list to sensitive proxy hosts.

### 7. Set Up Proxy Hosts in NPM (phpMyAdmin + wg-easy UI)

Create these proxy hosts **with SSL** (Let's Encrypt):

**A. phpMyAdmin**
- Domain: `pma.yourdomain.com` (or whatever you want)
- Forward Hostname / IP: `phpmyadmin`
- Forward Port: `80`
- Enable **SSL** → Request new Let's Encrypt certificate
- **Access List**: Select the one you created in step 6
- Advanced → enable "Block Common Exploits" (recommended)
- Save

**B. wg-easy Web UI**
- Domain: `wg.yourdomain.com`
- Forward Hostname / IP: `wg-easy`
- Forward Port: `51821`   (or whatever port wg-easy is using internally)
- Enable **SSL**
- **Access List**: Your admin list
- Save

**C. (Optional) NPM Admin UI itself**
- Domain: `npm.yourdomain.com`
- Forward to: `npm` on port `81`
- SSL + Access List

Once these are created and certificates issue successfully, you can access everything securely via your domains.

### 8. Set Up WireGuard VPN (wg-easy)

1. Access the wg-easy UI via the proxy host you just created (`wg.yourdomain.com`) or temporarily via `http://YOUR-SERVER-IP:51821` (if you mapped the port).
2. Log in with the `WG_PASSWORD` you set in `.env`.
3. Create your first client(s).
4. Download the config file or scan the QR code with the official WireGuard app on your phone/desktop.
5. Connect to the VPN.
6. **Note the client IP range** shown in the UI (usually starts with `10.8.0.x`). Add this subnet to your NPM Access List if not already done.

Now you can connect from anywhere via WireGuard and access your admin interfaces (even if your home IP changes).

### 9. Secure Everything Further (Recommended)

- In your server's firewall, restrict direct access to port 81 (NPM admin) to trusted IPs only, or remove the mapping later once proxied.
- Regularly update images:
  ```bash
  docker compose pull && docker compose up -d
  ```
- Back up your `./data/` folders and MariaDB (use `mysqldump` or tools like restic).
- Monitor logs:
  ```bash
  docker compose logs -f
  ```

### 10. Adding New Projects Later (Versatility)

New Docker projects can easily use NPM for SSL/proxying:

1. In a new project's `docker-compose.yml`, add:
   ```yaml
   networks:
     - homelab-proxy
   ```
2. At the bottom:
   ```yaml
   networks:
     homelab-proxy:
       external: true
   ```
3. In NPM, create a new proxy host pointing to your new container's name and port.
4. Apply SSL and (if admin) your Access List.

The shared `homelab-proxy` network makes service discovery trivial (`http://your-new-service:port`).

You can also attach new services to `homelab-backend` if they need direct MariaDB access (create limited DB users via phpMyAdmin — never use root for apps).

---

## File Overview

- `docker-compose.yml` — Main stack (heavily commented)
- `.env.example` — Template with **ALL CAPS warnings** on fields you must change
- `data/` folders — Created automatically or manually for persistence

---

## Security Highlights

- MariaDB not exposed publicly
- Admin UIs protected by NPM Access Lists (IP whitelist)
- WireGuard provides encrypted remote access
- Docker network isolation (`homelab-backend` for DB)
- All external traffic goes through NPM + SSL
- Each component has its own authentication layer on top

---

## Troubleshooting

- **Certificates not issuing?** → DNS must point to your server. Use HTTP challenge first.
- **Can't reach phpMyAdmin / wg-easy?** → Check NPM proxy host settings and Access List. Also verify containers are on the correct networks (`docker network inspect homelab-proxy`).
- **WireGuard clients can't reach internal services?** → Usually works out of the box. Check routes or try accessing via domain names through the tunnel.
- **Port conflicts?** → Change host ports in compose if needed.
- View logs for a service:
  ```bash
  docker compose logs -f npm
  docker compose logs -f wg-easy
  ```

---

## Next Steps & Customization

- Add more services (Nextcloud, Vaultwarden, etc.) using the shared proxy network.
- Consider adding Watchtower or Renovate for automatic updates.
- Set up proper backups.
- Explore advanced NPM features (redirects, custom locations, streaming, etc.).

---

**This setup gives you a professional-grade, secure base that is easy to grow.**

If you run into issues or want to extend it (add more services, change networking, etc.), feel free to open an issue or discuss.

Happy homelabbing! 
