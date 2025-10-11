# Remote Automation Hub SOP

## Overview
This Standard Operating Procedure (SOP) describes how to build a secure automation hub on an Ubuntu 22.04 VPS that hosts **n8n**, **Traefik**, and a **Cloudflare Tunnel**. The stack integrates with a Proxmox-based home lab to deliver monitoring, alerting, and self-healing automations while providing a reusable SaaS-ready blueprint.

## Repository Layout
```text
automation-hub/
├── .env                # Environment variables (copy & customise before use)
├── docker-compose.yml  # Core stack definition
├── n8n.env             # n8n-specific secrets & integration tokens
└── traefik/
    ├── acme.json       # LetsEncrypt certificate store (chmod 600)
    ├── dynamic.yml     # Middlewares and dashboard routers
    └── traefik.yml     # Static Traefik configuration
```

## Prerequisites
1. Ubuntu Server 22.04 LTS VPS with sudo access and ports **80/443** available.
2. Cloudflare account with a managed zone (e.g., `sylvect.biz`).
3. Proxmox home lab reachable through private networking or VPN (Tailscale/Cloudflare).
4. Docker 24.x and Docker Compose plugin.
5. Generated credentials:
   - Cloudflare API token (Zone:DNS edit + Zone:Zone read).
   - Cloudflare Tunnel token created via `cloudflared tunnel create`.
   - Traefik & n8n basic-auth hashes from `htpasswd`.
   - 32-character `N8N_ENCRYPTION_KEY`.

## Step-by-Step Deployment

### Step 1 — Harden & prepare the VPS
1. Update packages and install helpers:
   ```bash
   sudo apt update && sudo apt upgrade -y
   sudo apt install -y ca-certificates curl gnupg git
   ```
2. Create a non-root user with sudo and configure SSH over Tailscale/Cloudflare (port 22 blocked publicly).
3. Enable uncomplicated firewall (UFW) rules for HTTP/HTTPS only:
   ```bash
   sudo ufw default deny incoming
   sudo ufw default allow outgoing
   sudo ufw allow 80/tcp
   sudo ufw allow 443/tcp
   sudo ufw enable
   ```

### Step 2 — Install Docker Engine & Compose
```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
newgrp docker
```

### Step 3 — Retrieve the automation toolkit
```bash
mkdir -p ~/automation-hub && cd ~/automation-hub
git clone https://github.com/<your-org>/automation-hub.git .
```
If you are deploying directly from this SOP, copy the provided files into this directory.

### Step 4 — Configure secrets and network values
1. Copy the templates if you want to preserve the originals:
   ```bash
   cp .env .env.backup
   cp n8n.env n8n.env.backup
   ```
2. Edit `.env` and `n8n.env` with your values. Example snippet:
   ```ini
   ACME_EMAIL=ops@sylvect.biz
   CLOUDFLARE_API_TOKEN=cf_api_token_with_dns_edit
   CLOUDFLARE_TUNNEL_TOKEN=eyJhIjoi...
   DOCKER_NETWORK_NAME=automation_net
   N8N_DOMAIN=automate.sylvect.biz
   N8N_BASIC_AUTH_USER=ops-admin
   N8N_BASIC_AUTH_PASSWORD=change_me_now
   ```
3. Generate htpasswd hashes for Traefik and n8n middleware:
   ```bash
   sudo apt install -y apache2-utils
   htpasswd -nBC 12 traefik-admin
   htpasswd -nBC 12 n8n-admin
   ```
4. Replace the placeholder hostnames (`automations.example.com`) and hashed credentials inside `traefik/dynamic.yml` with the values generated in the previous step.
5. Set secure permissions:
   ```bash
   chmod 600 traefik/acme.json
   chmod 600 .env n8n.env
   ```

### Step 5 — Deploy the stack
```bash
# Ensure the Docker network matches DOCKER_NETWORK_NAME
export $(grep -v '^#' .env | xargs) && \
  docker network create "$DOCKER_NETWORK_NAME" 2>/dev/null || true

# Launch services
docker compose up -d
```

### Step 6 — Validate Traefik & Cloudflare Tunnel
1. Confirm containers are healthy:
   ```bash
   docker compose ps
   docker logs traefik -f
   docker logs cloudflared -f
   ```
2. In Cloudflare Zero Trust, map public hostnames (e.g., `automate.sylvect.biz`) to the internal origin `https://traefik:443` and enable TLS verification.
3. Visit `https://automate.sylvect.biz` to access n8n with basic auth and verify HTTPS lock. Check `https://traefik.automate.sylvect.biz` for the Traefik dashboard.

### Step 7 — Integrate with the Proxmox home lab
1. Connect the VPS to the home lab via Tailscale or WireGuard to reach VLAN segments.
2. Within n8n, create credentials for:
   - Proxmox API (`https://proxmox.example.local:8006/api2/json`).
   - Docker hosts (SSH or Docker Remote API via Tailscale).
   - Monitoring sources (Prometheus, Zabbix, custom scripts).
3. Build workflows for automated remediation:
   - Trigger on Proxmox events (VM stopped → send Telegram alert → auto-start LXC).
   - Monitor Docker Compose healthchecks via `HTTP Request` node → restart containers through SSH commands wrapped in n8n `Execute Command` nodes.
4. Store reusable JSON workflow exports under version control.

### Step 8 — Add external SaaS integrations
- **Notion:** Use the official Notion node with `NOTION_API_KEY` from `n8n.env` to log runbooks and incident updates.
- **Telegram:** Configure a Telegram bot for alerts and approvals.
- **GitHub:** Automate repository tasks (issue triage, PR reminders) via the GitHub node.
- **TradingView:** Accept webhook alerts through a public n8n webhook URL secured with the `TRADINGVIEW_WEBHOOK_SECRET` header.

### Step 9 — Monitoring, backups, and maintenance
1. **Observability**
   ```bash
   docker compose logs --since=1h traefik
   docker compose exec n8n n8n status
   ```
2. **Backups**
   - Snapshot the VPS volume weekly.
   - Export n8n workflows: *Settings → Workflow Export*. Store in Git.
   - Back up `n8n_data` volume:
     ```bash
     docker run --rm \
       -v n8n_data:/data \
       -v $(pwd)/backups:/backup \
       alpine tar czf /backup/n8n_data_$(date +%F).tgz -C /data .
     ```
3. **Patching**
   ```bash
   docker compose pull
   docker compose up -d
   sudo apt update && sudo apt upgrade -y
   ```
4. **Security checks**
   - Rotate API tokens quarterly.
   - Review `docker scout cves` and `trivy image` reports for base images.

### Step 10 — Scaling to multi-tenant SaaS
1. **Namespace isolation**: create dedicated Docker networks and `.env` files per client (e.g., `automation_net_client1`).
2. **n8n multi-instance**: run multiple n8n containers with unique subdomains and queues (Redis + Postgres backend for horizontal scaling).
3. **Central secrets management**: integrate HashiCorp Vault or Doppler; mount secrets into containers via environment files or sidecars.
4. **Observability stack**: ship logs to Loki + Grafana; expose metrics via Traefik Prometheus integration.
5. **CI/CD**: push configuration changes through GitHub Actions that lint YAML (`yamllint`), validate Compose files, and trigger `docker compose pull && up` via SSH.

## Verification Checklist
- [ ] Traefik dashboard reachable over HTTPS with auth.
- [ ] n8n admin panel protected by basic auth and TLS.
- [ ] Cloudflare Tunnel connected and hostname routes active.
- [ ] Home lab Proxmox API reachable from VPS.
- [ ] Automated workflows successfully trigger alerts and remediation.

## References
- [n8n Self-Hosting Docs](https://docs.n8n.io/hosting/)
- [Traefik v2 Documentation](https://doc.traefik.io/traefik/)
- [Cloudflare Tunnel Guide](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
