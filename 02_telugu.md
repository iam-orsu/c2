# Day 3: Setting Up Your Domain Redirector on Hostinger & Azure (YouTube Script in Tenglish)

---

## 🎙️ Video Intro & Recap

welcome back to 3rd video
of our series!

last video lo manam backend C2 server deploy chesam
 NAT Gateway attach chesi, Sliver install chesi, http listneer ni kuda set chesam


so today mission enti ante:
**building the Domain Redirector!**

Deeiniki pre requiste enti ante oka domain purchase chesi undali
meeku 100 ruppes lopu chala domains available lo untai get one

---

## 🖥️ Step 1: Deploy Redirector VM on Azure First

why first VM deploy chesthunnam ante:
manaki first Redirector Public IP ravali. 

aah IP vachina tharwathey DNS lo record mapping cheyagalam!

so let's spin up the VM in Azure Portal:

1. Search bar lo **Virtual machines** -> **+ Create** -> **Azure virtual machine**.
2. **Basics Tab**:
   * Resource group: `c2-infra-rg` (same group as C2 server!)
   * VM Name: `redirector`
   * Region: `India` (or same region as c2-vnet)
   * Image: `Ubuntu Server 22.04 LTS`
   * Size: `Standard_B2s`
   * Authentication: `Password` (username `c2admin`)
   * **Public inbound ports**: Select **None**!
     *(why NONE ante: custom network security group create chesi exact ports matrame allow chestham)*.
3. **Networking Tab**:
   * Virtual network: `c2-vnet`
   * Subnet: `default`
   * Public IP: Create new static IP named `redirector-pip`.
   * NIC Network Security Group: Select **Advanced** -> Create/select `redirector-nsg`.
4. Click **Review + create** -> **Create**.

VM deployed ayyaka overview page lo unna `REDIRECTOR_PUBLIC_IP` (e.g. `40.81.246.141`) note cheskondi!

---

## 🌐 Step 2: Configure Hostinger DNS A Record

ipudu manaki Redirector Public IP vachindhi kabatti, Hostinger lo DNS record map cheddam.
Nenu domain puchase ki hostinger vadthunna,
But vere platfrom like go daddy, Name cheap lo kuda almost same

**idhi endhuku chesthunnam ante:**
oka implant ggenerate chesthunnapudu payload lo direct c2 IP ivvakunda 
domain name (`updates.ceruleanpay.online`) istham.

so internet lo una DNS systems e domain ni mana Redirector VM IP ki point chesthai.

1. Open **Hostinger Dashboard** -> go to **DNS / Nameservers**.
2. Create new A Record:
   * **Type**: `A`
   * **Name / Subdomain**: `updates` (so full hostname becomes `updates.ceruleanpay.online`)
   * **Points to / Value**: Enter your `REDIRECTOR_PUBLIC_IP` (apudey note cheskunna IP)
   * **TTL**: `300` (5 minutes fast DNS propagation kosam)
3. Click **Add Record**.

---

## 🔒 Step 3: Configure Network Security Group (`redirector-nsg`)

Redirector public-facing website kabatti, 
daniki exactly required ports matrame open చేయాలి.

so go to **Network security groups** -> `redirector-nsg` -> **Inbound security rules** -> 

oka 3 rules add cheddam

### Rule 1: Allow SSH (Port 22)
* Source: `IP Addresses` -> enter your operator IP (`curl.exe -4 ifconfig.me` lo vachina IP)
* Destination Port: `22`, Protocol: `TCP`, Action: `Allow`, Priority: `100`, Name: `Allow-SSH-OperatorOnly`
* *(why? so that only we can SSH into the redirector VM!)*

### Rule 2: Allow HTTP (Port 80)
* Source: `Any`
* Destination Port: `80`, Protocol: `TCP`, Action: `Allow`, Priority: `110`, Name: `Allow-HTTP-Public`
* *(why? Certbot Let's Encrypt SSL validation servers Port 80 lo domain ownership test chesthai!)*

### Rule 3: Allow HTTPS (Port 443)
* Source: `Any`
* Destination Port: `443`, Protocol: `TCP`, Action: `Allow`, Priority: `120`, Name: `Allow-HTTPS-Public`
* *(why? target beacon implant traffic encrypted HTTPS Port 443 lo vasthundhi!)*

Perfect alldone
---

## 🔗 Step 4: Update C2 Server NSG (`c2-server-nsg`)

now super critical OPSEC step!

mana backend C2 server port 8443 ni entire internet ki block chesi, **only Redirector VM ki matrame allow chestham.**
Very very important step! Don't mess up here.

**endhuku chesthunnam ante:**
internet lo unna evadaina mana C2 server IP ni direct ga port scan cheste, 
connection drop aypothadhi. So our C2 server will be completely invisible!

1. Go to Azure Portal -> **Network security groups** -> `c2-server-nsg`.
2. Click **Inbound security rules** -> **+ Add**:
   * Source: `IP Addresses`
   * Source IP: enter your `REDIRECTOR_PUBLIC_IP` (or VNet private IP `10.0.0.5`)
   * Destination Port: `8443`
   * Protocol: `TCP`, Action: `Allow`, Priority: `110`, Name: `Allow-C2-FromRedirector`
3. Click **Add**.

---

## 📦 Step 5: Install Nginx & Certbot on Redirector

now VM lo SSH login avvandi:

```bash
ssh c2admin@REDIRECTOR_PUBLIC_IP
```

Nginx and cerbot install cheddam

**endhuku Nginx & Certbot ante:**
Nginx reverse proxy ga work incoming requests ni filter chesthundhi. 
and Certbot free SSL certificates generate chesthundhi 

so our our webiste will have https and traffic encrypted ga untadhi.

system packages update chesi install cheddam:

```bash
sudo apt update && sudo apt install -y nginx certbot python3-certbot-nginx
```

---

## 🔐 Step 6: Issue SSL Certificate via Certbot

ssl certificate generate cheddam

**why stop Nginx first?**
First niginx ni stop cheddam 
Nginx already Port 80 use chesthu untadhi. 
so that cert bot binding will work
```bash
sudo systemctl stop nginx
```

now run certbot command:

```bash
sudo certbot certonly --standalone -d updates.ceruleanpay.online --register-unsafely-without-email --agree-tos
```

`Successfully received certificate` ani vasthey **SUCCESS**!
SSL certificates `/etc/letsencrypt/live/updates.ceruleanpay.online/` lo save avthai.

---

## 🌐 Step 7: Create Decoy Landing Page

**why Decoy Page?**
if Blue Team analyst or Shodan/Censys scanners `updates.ceruleanpay.online` crawl open cheste, 
suspicious error or empty page raakunda, 
real merchant website site `CeruleanPay Gateway` kanipisthundhi. 

so normal business portal anukoni ignore chestharu!

create decoy HTML file:

```bash
sudo nano /var/www/html/index.html
```

paste this fake merchant gateway HTML:

```html
<!DOCTYPE html>
<html>
<head>
    <title>CeruleanPay Gateway</title>
    <style>
        body { font-family: Arial, sans-serif; text-align: center; padding-top: 100px; background-color: #f4f6f9; }
        .card { background: white; padding: 40px; border-radius: 8px; display: inline-block; box-shadow: 0 4px 6px rgba(0,0,0,0.1); }
        h1 { color: #1a56db; }
        p { color: #555; }
    </style>
</head>
<body>
    <div class="card">
        <h1>CeruleanPay Gateway</h1>
        <p>Merchant Payment System v2.4 Active</p>
        <p>Status: All Systems Operational</p>
    </div>
</body>
</html>
```

Save (`Ctrl+O`, `Enter`, `Ctrl+X`).

---

## ⚙️ Step 8: Configure Nginx Reverse Proxy with Secret Path

now main core configuration!

create Nginx site configuration file:
First paste chesi meeku explain chestha
```bash
sudo nano /etc/nginx/sites-available/updates.ceruleanpay.online
```

paste this production configuration:

```nginx
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

    if ($blocked_agent) {
        return 404;
    }

    # Secret C2 Path Forwarding (Proxy Pass to C2 Private IP)
    location /updates/v2/sync {
        proxy_pass https://10.0.0.4:8443;
        proxy_ssl_verify off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }

    # Default Decoy Site
    location / {
        root /var/www/html;
        index index.html;
    }
}
```
**endhuku Scanner Blocking & Secret Path ante:**
1. **Scanner Blocking (`$blocked_agent`)**: `Nmap`, `Nikto`, `Sqlmap`, `Shodan`, `Censys` 
lanti tools hit cheste Nginx instantly 404 return chesi block chesthundhi.

2. **Secret Path (`/updates/v2/sync`)**: Only target beacon payload ki matrame e path telusthundhi. 
e path lo oche traffic matrame backend C2 private IP `10.0.0.4:8443` ki proxy pass avthundhi!

Save (`Ctrl+O`, `Enter`, `Ctrl+X`).

now site enable chesi default Nginx config remove cheddam:

```bash
sudo ln -s /etc/nginx/sites-available/updates.ceruleanpay.online /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
```

test Nginx config syntax:

```bash
sudo nginx -t
```

`syntax is ok` vasthey restart Nginx:

```bash
sudo systemctl restart nginx
```

---

## 🧪 Step 9: Testing & Verification

now let's verify all 3 security layers!

### Test 1: Decoy Site Verification
browser lo `https://updates.ceruleanpay.online` open cheyyandi.
"CeruleanPay Gateway - Merchant Payment System v2.4 Active" decoy page kanipistundhi!

### Test 2: Scanner Blocking Verification
local terminal nundi scanner user-agent pass chesi test cheddam:
```bash
curl -kv -A "Nmap" https://updates.ceruleanpay.online
```
Nginx instantly returns `HTTP/1.1 404 Not Found`! Scanner blocked!

### Test 3: Secret C2 Path Proxy Verification
local terminal nundi browser user-agent & secret path test cheddam:
```bash
curl -kv -H "User-Agent: Mozilla/5.0" https://updates.ceruleanpay.online/updates/v2/sync
```
output lo `< HTTP/1.1 404 Not Found` from Sliver vastundi! 
that means Nginx secret path request ni successfully backend C2 server `10.0.0.4:8443` ki proxy pass chesindhi!

---

## 🎬 Outro

boooom! Day 3 mission complete!

let's recap what we built today:
* ✅ Provisioned `redirector` VM on Azure first to get its Public IP.
* ✅ Mapped Hostinger DNS A record (`updates.ceruleanpay.online`) to the Redirector IP.
* ✅ Secured C2 Server port 8443 to only accept traffic from Redirector.
* ✅ Issued valid Let's Encrypt SSL certificate via Certbot.
* ✅ Deployed merchant decoy landing page & Nginx scanner blocking rules.
* ✅ Configured Nginx secret path proxy forwarding (`/updates/v2/sync`) to backend C2 on `10.0.0.4:8443`.

our domain redirector pipeline is now **100% live, encrypted, and OPSEC-hardened!**

in **Day 4 (Final Video)**, manam Sliver C2 lo Windows payload compile chesi, Windows target machine lo run chesi, live beacon session obtain chesi full network telemetry verify cheddam!

so yeah don't forget to **Like** and **Subscribe**!

see you in Day 4! ✌️
