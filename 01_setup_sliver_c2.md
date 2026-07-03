# Day 2: Setting Up Your Sliver C2 Server on Azure

---

## Overview & Architecture

Today we deploy and harden the C2 backend server (`c2-server`). 

```
[ OPERATOR IP ] ---> SSH (Port 22) ---> [ C2 SERVER VM ]
                                              |
                                     Sliver HTTPS Listener
                                          (Port 8443)
```

### VM Specifications & Network Plan
* **VM Name**: `c2-server`
* **OS**: Ubuntu Server 22.04 LTS
* **Size**: Standard_B2s (2 vCPU, 4 GiB RAM)
* **Resource Group**: `c2-infra-rg`
* **VNet / Subnet**: `c2-vnet` / `c2-subnet` (10.0.0.0/24)
* **Inbound Access**:
  * Port 22 (SSH): Locked strictly to `YOUR_OPERATOR_IP`
  * Port 8443 (Sliver HTTPS): Allowed ONLY from `REDIRECTOR_IP`
* **Outbound Access**: Routed via Azure NAT Gateway (`c2-nat-gateway`)

---

## Step 1: Discover Your Operator Public IP

> **Intent**: Finds your local workstation's public IP address so you can lock down SSH access strictly to your machine.

On your local workstation terminal, run:

```bash
curl -s https://ifconfig.me
```

* **Output**: Your public IPv4 address (e.g., `157.50.74.229`).
* **Note**: We will refer to this as `YOUR_OPERATOR_IP`.

---

## Step 2: Create Azure Resource Group & VNet

> **Intent**: Creates the cloud container and private virtual network foundation for all C2 infrastructure.

1. Log in to [portal.azure.com](https://portal.azure.com).
2. Search for **Resource groups** -> Click **+ Create**.
   * **Resource group**: `c2-infra-rg`
   * **Region**: `East US` (or your preferred region) -> Click **Review + create** -> **Create**.
3. Search for **Virtual networks** -> Click **+ Create**.
   * **Resource group**: `c2-infra-rg`
   * **Name**: `c2-vnet`
   * **Region**: `East US`
4. Go to **Address space** tab:
   * VNet Address Space: `10.0.0.0/16`
   * In **Subnets** list, edit default subnet:
     * **Name**: `c2-subnet`
     * **Starting address**: `10.0.0.0`, **Size**: `/24 (256 addresses)`
     * **Enable private subnet (no default outbound access)**: Check this box (March 2026 default).
   * Click **Save** -> **Review + create** -> **Create**.

---

## Step 3: Attach Azure NAT Gateway (Required for Outbound Internet)

> **Intent**: Gives our private VNet subnet outbound internet access so `apt update` and Sliver installation scripts work.

Since Azure VNet subnets created after March 2026 block implicit outbound access by default, we attach a NAT Gateway so the VM can download `apt` packages and Sliver dependencies.

1. Search for **NAT gateways** -> Click **+ Create**.
   * **Resource group**: `c2-infra-rg`
   * **Name**: `c2-nat-gateway`
   * **Region**: `East US`
   * **SKU**: `Standard V2 (Recommended)`
2. Go to **Outbound IP** tab:
   * Click **+ Add public IP addresses or prefixes** -> Click **Create a public IP address**.
   * **Name**: `c2-nat-pip` -> Click **OK**.
3. Go to **Networking** tab:
   * **Virtual network**: Select `c2-vnet` from dropdown.
   * **Subnets**: Check `default (10.0.0.0/24)` (or `c2-subnet` if you renamed it).
4. Click **Review + create** -> **Create**.

---

## Step 4: Create C2 Network Security Group (NSG)

> **Intent**: Sets up the cloud-level firewall to restrict inbound SSH traffic exclusively to your operator IP.

1. Search for **Network security groups** -> Click **+ Create**.
   * **Resource group**: `c2-infra-rg`
   * **Name**: `c2-server-nsg` -> Click **Review + create** -> **Create**.
2. Navigate to `c2-server-nsg` -> **Settings** -> **Inbound security rules** -> Click **+ Add**.
   * **Source**: `IP Addresses`
   * **Source IP addresses**: `YOUR_OPERATOR_IP` (e.g. `157.50.74.229` plainly, Azure automatically treats single host IPs as `/32`).
   * **Source port ranges**: `*`
   * **Destination**: `Any`
   * **Service**: `SSH`
   * **Destination port ranges**: `22`
   * **Protocol**: `TCP`
   * **Action**: `Allow`
   * **Priority**: `100`
   * **Name**: `Allow-SSH-OperatorOnly`
   * Click **Add**.

---

## Step 5: Provision the C2 Server VM

> **Intent**: Deploys the Ubuntu 22.04 virtual machine that acts as the backend C2 server inside your private subnet.

1. Search for **Virtual machines** -> Click **+ Create** -> **Azure virtual machine**.
2. **Basics Tab**:
   * **Resource group**: `c2-infra-rg`
   * **Virtual machine name**: `c2-server`
   * **Region**: `East US`
   * **Image**: `Ubuntu Server 22.04 LTS - x64 Gen2`
   * **Size**: `Standard_B2s`
   * **Authentication type**: `Password`
   * **Username**: `c2admin`
   * **Password**: *[Set a strong password]*
   * **Public inbound ports**: Select **None** (do NOT select "Allow selected ports", as that opens SSH to the entire world).
3. **Networking Tab**:
   * **Virtual network**: `c2-vnet`
   * **Subnet**: `c2-subnet` (or `default`)
   * **Public IP**: Click **Create new** -> Name: `c2-server-pip`, Assignment: `Static` -> **OK**.
   * **NIC network security group**: Select **Advanced** (or **Configure**) -> Select **`c2-server-nsg`**.
4. **Monitoring Tab**:
   * **Boot diagnostics**: Set to `Enable with managed storage account`. (Enables Azure Serial Console emergency access).
5. Click **Review + create** -> **Create**.
6. **Save Public IP**: Once deployed, note down `C2_SERVER_IP` from the VM overview page.

---

## Step 6: Initial Hardening & Firewall (fail2ban + UFW)

> **Intent**: Installs OS-level brute-force protection (`fail2ban`) and local firewall (`ufw`) as a second layer of defense.

SSH into `c2-server` from your workstation:

```bash
ssh c2admin@C2_SERVER_IP
```

### 1. Verify Outbound Internet Access via NAT Gateway
```bash
ping -c 4 8.8.8.8
```
* **Expected Output**: 4 packets transmitted, 4 received (0% packet loss).

### 2. Enable Universe Repository & Update System Packages
```bash
sudo add-apt-repository universe -y && sudo apt update && sudo apt upgrade -y
```

*(Note: Enabling `universe` is required on Ubuntu 22.04 LTS so `apt` can locate the `minisign` dependency used by Sliver).*

### 3. Install & Configure fail2ban
```bash
sudo apt install -y fail2ban
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```
Add/modify under `[DEFAULT]` and `[sshd]`:
```ini
[DEFAULT]
ignoreip = 127.0.0.1/8 ::1 YOUR_OPERATOR_IP
bantime = 86400
findtime = 600
maxretry = 3

[sshd]
enabled = true
mode = aggressive
port = ssh
```
Save and restart service:
```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd
```

### 4. Enable UFW Host Firewall
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from YOUR_OPERATOR_IP to any port 22 proto tcp
sudo ufw allow 8443/tcp
sudo ufw enable
```
Type `y` when prompted. Verify status:
```bash
sudo ufw status numbered
```
* **Expected Output**:
```
Status: active

     To                         Action      From
     --                         ------      ----
[ 1] 22/tcp                     ALLOW IN    YOUR_OPERATOR_IP
[ 2] 8443/tcp                   ALLOW IN    Anywhere
```

---

## Step 7: Install Sliver C2 & Configure Service

### 1. Install Minisign Dependency (Ubuntu 22.04 Fix)
Since `minisign` is omitted from default Ubuntu 22.04 apt repositories, install the binary manually to `/usr/local/bin`:

```bash
curl -sSL https://github.com/jedisct1/minisign/releases/download/0.11/minisign-0.11-linux.tar.gz | tar xz
sudo mv minisign-linux/x86_64/minisign /usr/local/bin/
sudo chmod +x /usr/local/bin/minisign
```

### 2. Run the Sliver Installer Script
Execute the official Sliver installer:

```bash
curl https://sliver.sh/install | sudo bash
```

* **Expected Output**:
```
[*] Downloading sliver-server...
[*] Installing mingw-w64 cross-compiler...
[*] Creating systemd service...
[*] Sliver installed to /usr/local/bin/sliver-server
```

### Enable & Start Sliver Systemd Service
```bash
sudo systemctl daemon-reload
sudo systemctl enable sliver
sudo systemctl start sliver
sudo systemctl status sliver
```
* **Expected Output**: `Active: active (running)`.

---

## Step 8: Start HTTPS Listener (Port 8443)

> **Intent**: Activates the internal C2 listener on port 8443 to receive proxied implant traffic from the redirector.

Launch the interactive Sliver console on `c2-server`:

```bash
sudo sliver
```

* **Expected Output**:
```
  ██████  ██▓     ██▓ ██▒   █▓▓█████  ██▀███
 ▒██    ▒ ▓██▒    ▓██▒▓██░   █▒▓█   ▀ ▓██ ▒ ██▒
 ░ ▓██▄   ▒██░    ▒██▒ ▓██  █▒░▒███   ▓██ ░▄█ ▒

[*] Server v1.5.x - Bishop Fox
sliver >
```

Start the internal HTTPS listener on port 8443:

```
sliver > https --lport 8443
```

* **Expected Output**:
```
[*] Starting HTTPS :8443 listener ...
[!] Successfully started job #1
```

Verify active job:

```
sliver > jobs
```

```
 ID   Name    Protocol   Port   Domains
==== ======== ========== ====== =========
 1    https   tcp        8443
```

---

## Step 9: Verify Listener Locally

> **Intent**: Tests locally on the C2 server that the HTTPS listener is running and responding properly before attaching the redirector.

Open a second SSH session to `c2-server` and test local connection to port 8443:

```bash
curl -kv https://127.0.0.1:8443
```

* **Expected Output**:
```
* Connected to 127.0.0.1 (127.0.0.1) port 8443 (#0)
< HTTP/1.1 404 Not Found
```
*(A 404 response confirms the listener is active and rejecting non-implant HTTP requests).*

---

## Day 2 Summary Checklist

| Component | Status | Verification Command / Result |
|---|---|---|
| NAT Gateway | Verified | `ping 8.8.8.8` returns replies |
| NSG Rules | Verified | Port 22 allowed from `YOUR_OPERATOR_IP` only |
| fail2ban | Active | `sudo fail2ban-client status sshd` running |
| UFW | Enabled | `sudo ufw status` active |
| Sliver Service | Active | `sudo systemctl status sliver` active (running) |
| HTTPS Listener | Active | `sliver > jobs` shows HTTPS on :8443 |

---
**Next Step**: Proceed to [02_domain_redirector.md](./02_domain_redirector.md) to set up Nginx and Certbot on the redirector.
