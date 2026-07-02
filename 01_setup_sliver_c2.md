# Day 1: Setting Up the Sliver C2 Server

## What We Are Building Today

"Day 1. You build the C2 server. By end of day Sliver runs on a hardened Ubuntu VM hidden from the internet. Only the WireGuard tunnel we build tomorrow provides a way in."

| Step | Task | Why |
|------|------|-----|
| 1 | Create Resource Group | Container for all resources. Delete this when engagement ends and everything disappears. |
| 2 | Create Virtual Network and Subnet | Private network for our VMs to talk to each other without internet exposure. |
| 3 | Create the VM inside that VNet | The actual server that runs Sliver. |
| 4 | Lock down NSG firewall | Block all inbound from internet. VM can download packages but nobody can reach it. |
| 5 | Connect to the VM | Get terminal access to configure the server. |
| 6 | Harden the VM | Lock down SSH, enable firewall, block brute force. Reduce attack surface. |
| 7 | Install Sliver | Install the C2 framework that manages implants. |
| 8 | Configure WireGuard | Set up encrypted tunnel keys so redirector can connect tomorrow. |
| 9 | Start HTTPS listener | Open a port for implants to call back to through the tunnel. |
| 10 | Enable multi-operator | Let the whole team connect to Sliver at the same time. |

---

## Azure 2026 Change

In April 2026 Azure made new VNets private by default. A VM without a public IP in a new VNet has **no outbound internet**. No apt. No GitHub. Nothing.

**Our fix:** Give the VM a public IP for setup so it can download packages. Lock it with NSG rules so nobody can connect IN. Remove the public IP later if you want. WireGuard handles everything incoming after setup.

---

## Step 1: Create Resource Group

Search bar → `Resource groups` → click → **+ Create**

| Field | Value | Why |
|-------|-------|-----|
| Subscription | Your subscription | - |
| Resource group | `rg-c2-infra` | All engagement resources in one place. Delete the group = delete everything. |
| Region | `East US` | Keep all resources in same region. Cross-region traffic adds latency. |

Click **Review + create** → **Create**

---

## Step 2: Create Virtual Network and Subnet

**Why we create this first:** When we make the VM later we just pick this VNet from a dropdown. No confusing subnet checkboxes buried in the VM wizard.

Search bar → `Virtual networks` → click → **+ Create**

**Basics tab:**

| Field | Value | Why |
|-------|-------|-----|
| Subscription | Your subscription | - |
| Resource group | `rg-c2-infra` | Same group as everything else. |
| Name | `vnet-c2-infra` | Identifiable name. We reference this when creating the VM. |
| Region | `East US` | Same region as Resource Group. |

**Security tab:** Leave defaults. We do not need Bastion or DDoS protection for a C2 server.

**IP Addresses tab:**

| Setting | Value | Why |
|---------|-------|-----|
| Address space | `10.0.0.0/16` (default) | Gives us 65,536 private IPs. Way more than we need but it is the default. |
| Subnet name | `default` | Default subnet is fine. |
| Subnet range | `10.0.0.0/24` (default) | 256 IPs for this subnet. Plenty for a few VMs. |

- If you see "Private subnet" or "Default outbound access" → make sure outbound is **enabled**. Otherwise VMs in this VNet cannot reach the internet even with a public IP.
- If you do not see this option → fine, Azure handled it for you.

Click **Review + create** → **Create**

---

## Step 3: Create the VM

**Why we pick the existing VNet:** Easier than creating everything in the VM wizard. We already decided the network layout. Now we just place a VM into it.

Search bar → `Virtual machines` → **+ Create** → **Azure virtual machine**

**Basics tab:**

| Field | Value | Why |
|-------|-------|-----|
| Subscription | Your subscription | - |
| Resource group | `rg-c2-infra` | - |
| Virtual machine name | `c2-server` | Identifiable name in the portal. |
| Region | `East US` | Same as VNet. VM and VNet must be in same region. |
| Availability options | No redundancy required | Single VM for a 2-week engagement. HA not needed. |
| Security type | Standard | Trusted Launch not needed for our use. |
| Image | Ubuntu Server 22.04 LTS - x64 Gen2 | 22.04 is stable, well-documented, supported until 2027. |
| Size | B1s (1 vCPU, 1 GB RAM, ~$7/month) | Sliver is lightweight. Does not need much CPU or RAM. |
| Authentication type | Password | Simple. VM is not internet-facing so brute force is not a concern. |
| Username | `operator` | Generic name. Does not indicate what the server does. |
| Password | Strong password. 16+ chars. Write it down. | Only way to log in. Lose this and you need the Azure console. |
| Public inbound ports | None | We create our own NSG rules. Never let Azure open ports automatically. |

**Disks tab:** Leave defaults. Standard SSD is fine. We are not doing heavy I/O.

**Networking tab:**

| Field | Value | Why |
|-------|-------|-----|
| Virtual network | `vnet-c2-infra` | The VNet we created in Step 2. |
| Subnet | `default` | The subnet inside that VNet. |
| Public IP | Create new → `c2-server-ip` → Standard SKU → OK | Gives VM outbound internet for package downloads. We lock it inbound in Step 4. |
| NIC network security group | Create new → `nsg-c2-server` → OK | Virtual firewall attached to the VM's network card. |
| Delete NIC when VM is deleted | Checked | Auto-cleanup. No orphaned resources after engagement. |
| Accelerated networking | Unchecked | Not needed. Low traffic C2 server. |
| Load balancing | None | Single VM. No load balancer needed. |

**Management, Monitoring, Advanced tabs:** Leave defaults. Boot diagnostics, auto-shutdown, backup not needed for a 2-week engagement.

**Review + create → Create.** Wait 2-3 minutes. Click **Go to resource**.

Write down from the overview page:

- Public IP: `___` - used to SSH in during setup
- Private IP: `___` - used tomorrow for WireGuard endpoint

---

## Step 4: Lock Down NSG Firewall

**Why:** The VM can now reach the internet to download packages. But we must block all inbound traffic so nobody on the internet can reach the VM.

VM page → left menu → **Networking** → inbound port rules.

**Rule 1 - Allow SSH from VNet only:**

Click **+ Add inbound port rule**

| Field | Value | Why |
|-------|-------|-----|
| Source | IP Addresses | Only allow specific IP ranges. |
| Source IP addresses | `10.0.0.0/16` | Our VNet address space. Only VMs inside our VNet can SSH. |
| Source port ranges | * | Any source port. |
| Destination | Any | - |
| Service | SSH | Port 22. |
| Action | Allow | - |
| Priority | 100 | Lower number = higher priority. This runs before the deny rule. |
| Name | AllowSSH-VNetOnly | Descriptive name so we know what this rule does. |

Click **Add**

**Rule 2 - Deny everything from internet:**

Click **+ Add inbound port rule**

| Field | Value | Why |
|-------|-------|-----|
| Source | Internet | All internet traffic. |
| Destination port ranges | * | Block every port. |
| Protocol | Any | Block TCP and UDP both. |
| Action | Deny | Drop the traffic. No response sent back. |
| Priority | 200 | Runs after Rule 1. Allow VNet SSH first, then deny everything else. |
| Name | DenyAllInternet | - |

Click **Add**

**Result:** VM can reach OUT to internet (apt, wget, GitHub). Nobody from internet can reach IN (SSH blocked, all ports blocked). VMs inside our VNet can still SSH.

---

## Step 5: Connect to the VM

**Why we need a workaround:** The NSG blocks inbound SSH from internet. We need a temporary hole or a side channel.

**Option A - Temporary SSH rule (fastest):**

- Go to `https://whatismyip.com` → copy your public IP
- VM page → Networking → **+ Add inbound port rule**

| Field | Value |
|-------|-------|
| Source | IP Addresses |
| Source IP addresses | `YOUR_IP/32` |
| Service | SSH |
| Action | Allow |
| Priority | 90 |
| Name | AllowSSH-MyLaptop |

- **Why priority 90:** Runs before the VNet rule (100) and the deny rule (200). Lets your specific IP through.
- Click Add
- `ssh operator@<VM_PUBLIC_IP>`
- Enter password
- **Delete this rule after setup.** Do not leave a hole in the firewall.

**Option B - Serial Console (free, always works):**

- VM page → Connect → Serial console
- Login with `operator` / password
- **Why this works:** Serial console connects directly to the VM's virtual serial port. Bypasses all network rules entirely. Works even if you completely break the firewall.

**Option C - Azure Bastion (browser SSH, costs extra):**

- VM page → Connect → Bastion → enter credentials
- **Why:** Browser-based SSH through Azure's managed service. No public IP needed on the VM at all.

---

## Step 6: Harden the VM

**Why:** Fresh Azure VM has default settings. We lock it down before installing anything.

### Update and Install Tools

```bash
sudo apt update && sudo apt upgrade -y
```

**Why:** Patches all known vulnerabilities in the base Ubuntu image. First thing on any new server.

```bash
sudo apt install -y curl wget ufw fail2ban wireguard
```

| Tool | Why we install it |
|------|-------------------|
| curl, wget | Download Sliver binaries from GitHub. |
| ufw | Simple firewall. Blocks everything by default, we open only what we need. |
| fail2ban | Watches auth logs. After 5 failed SSH attempts it bans the IP for 1 hour. Slows brute force. |
| wireguard | Creates encrypted tunnel between this server and the redirector. |

### Configure SSH

```bash
sudo nano /etc/ssh/sshd_config
```

Find each line. Remove `#` if commented out. Add if missing.

```
PermitRootLogin no
PasswordAuthentication yes
PubkeyAuthentication yes
KbdInteractiveAuthentication no
ChallengeResponseAuthentication no
MaxAuthTries 3
LoginGraceTime 30
X11Forwarding no
PermitEmptyPasswords no
AllowUsers operator
```

| Setting | Value | Why |
|---------|-------|-----|
| PermitRootLogin | no | Root cannot SSH directly. Must use operator + sudo. Prevents direct root brute force. |
| PasswordAuthentication | yes | Allow password login. This VM is behind NSG so internet cannot brute force it. |
| MaxAuthTries | 3 | After 3 wrong passwords SSH disconnects. Attacker must reconnect to try again. |
| LoginGraceTime | 30 | 30 seconds to authenticate or SSH disconnects. Wastes attacker time. |
| X11Forwarding | no | No graphical forwarding. Reduces attack surface. |
| PermitEmptyPasswords | no | Accounts with no password cannot log in. |
| AllowUsers | operator | Only the operator user can SSH. All other users blocked even with correct password. |

```bash
sudo sshd -t       # Check for syntax errors. No output = valid.
sudo systemctl reload ssh   # Apply changes without dropping current session.
```

### Configure UFW Firewall

```bash
sudo ufw default deny incoming
```

**Why:** Block all inbound traffic by default. We open only specific ports from specific sources.

```bash
sudo ufw default allow outgoing
```

**Why:** VM needs to make outbound connections to download packages, reach GitHub, and eventually communicate through WireGuard.

```bash
sudo ufw allow from 10.0.0.0/16 to any port 22 proto tcp
```

**Why:** Allow SSH only from inside our Azure VNet. Internet cannot SSH even if NSG somehow failed.

```bash
sudo ufw allow 51820/udp
```

**Why:** WireGuard uses UDP 51820. Open this so the redirector can establish the tunnel tomorrow.

```bash
sudo ufw --force enable
sudo ufw status verbose
```

Expected output:

```
Status: active
Default: deny (incoming), allow (outgoing)
22/tcp                     ALLOW IN    10.0.0.0/16
51820/udp                  ALLOW IN    Anywhere
```

NSG blocks at Azure network level. UFW blocks at OS level. Two layers of firewall. If one fails, the other still protects.

### Configure Fail2ban

```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

**Why:** Copy default config. `.local` file overrides `.conf`. Updates do not erase our settings.

```bash
sudo nano /etc/fail2ban/jail.local
```

In `[DEFAULT]`:

```
bantime = 3600
findtime = 600
maxretry = 5
ignoreip = 127.0.0.1/8 10.0.0.0/16
```

| Setting | Value | Why |
|---------|-------|-----|
| bantime | 3600 | Ban IP for 1 hour (3600 seconds). |
| findtime | 600 | Count failures within 10 minutes (600 seconds). |
| maxretry | 5 | 5 failures in 10 minutes = ban. |
| ignoreip | 127.0.0.1/8 10.0.0.0/16 | Never ban localhost or our own VNet IPs. |

In `[sshd]`:

```
[sshd]
enabled = true
mode = aggressive
```

**Why aggressive mode:** Enables extra filters that catch more types of SSH attacks, not just password failures.

```bash
sudo systemctl enable fail2ban    # Auto-start on boot
sudo systemctl start fail2ban     # Start now
sudo systemctl status fail2ban    # Verify running
```

---

## Step 7: Install Sliver

### Download Binaries

```bash
sudo mkdir -p /opt/sliver
cd /opt/sliver
```

**Why `/opt/sliver`:** Standard Linux location for optional/third-party software packages.

```bash
sudo wget https://github.com/BishopFox/sliver/releases/latest/download/sliver-server_linux -O sliver-server
sudo wget https://github.com/BishopFox/sliver/releases/latest/download/sliver-client_linux -O sliver-client
```

**Why two binaries:** Server runs as daemon on this VM. Client is the command-line tool operators use to connect to the server. We install both here.

```bash
sudo chmod +x sliver-server sliver-client
```

**Why:** Downloaded files are not executable by default. chmod +x makes them runnable.

### Install Build Dependencies

```bash
sudo apt install -y mingw-w64 binutils-mingw-w64 g++-mingw-w64
```

**Why:** MinGW cross-compiler lets Linux build executables that run on Windows. Sliver needs this to generate Windows implants. Without it, you can only generate Linux implants.

### Unpack Assets

```bash
sudo /opt/sliver/sliver-server unpack --force
```

**Why:** First-run setup. Downloads Go toolchain, compiles default implant templates for Windows/Linux/macOS. Takes 2-3 minutes. Only needed once.

Expected output:

```
[i] Unpacking Sliver assets...
[i] Downloading Go toolchain...
[i] Compiling default implant templates...
[i] Unpacking complete!
```

### Systemd Service

```bash
sudo nano /etc/systemd/system/sliver.service
```

**Why systemd:** Makes Sliver run in the background. Auto-starts on boot. Auto-restarts if it crashes. Standard way to run services on Ubuntu.

```ini
[Unit]
Description=Sliver C2 Server
After=network.target

[Service]
Type=simple
User=root
Group=root
WorkingDirectory=/opt/sliver
ExecStart=/opt/sliver/sliver-server daemon
Restart=always
RestartSec=5
LimitNOFILE=50000

[Install]
WantedBy=multi-user.target
```

| Setting | Why |
|---------|-----|
| After=network.target | Wait until network is ready. Sliver needs network to start listeners. |
| User=root | Sliver binds to network ports. Only root can bind ports below 1024. |
| ExecStart=...daemon | Runs Sliver in background daemon mode. Accepts client connections. |
| Restart=always | If Sliver crashes or exits, systemd restarts it. Keeps C2 always available. |
| RestartSec=5 | Wait 5 seconds before restarting. Prevents rapid restart loops. |
| LimitNOFILE=50000 | Increase open file limit. Sliver maintains many concurrent connections. |

```bash
sudo systemctl daemon-reload   # Systemd reads the new service file
sudo systemctl enable sliver   # Auto-start on boot
sudo systemctl start sliver    # Start now
sudo systemctl status sliver   # Check running
```

Must show `active (running)`.

### Connect and Verify

```bash
sudo ln -s /opt/sliver/sliver-client /usr/local/bin/sliver
```

**Why symlink:** Put `sliver` in PATH so you can type `sliver` from anywhere instead of `/opt/sliver/sliver-client`.

```bash
sliver
```

```
sliver > version
```

**Why:** Confirm client and server are same version. Mismatched versions cause bugs.

```
sliver > armory install all
```

**Why:** Armory adds pre-built tools for credential dumping, AD enumeration, lateral movement. Install now so they are available during the engagement.

```
sliver > exit
```

---

## Step 8: Configure WireGuard

**Why WireGuard:** Creates encrypted tunnel between redirector (tomorrow) and this C2 server. The C2 server has no open ports to the internet. Only way in is through this tunnel.

### Generate Keys

```bash
sudo mkdir -p /etc/wireguard
cd /etc/wireguard
sudo umask 077
```

**Why umask 077:** Files created in this directory are only readable by root. Protects the private key.

```bash
wg genkey | sudo tee privatekey
```

**Why:** Generates a random 44-character base64 private key. This is the secret half of the keypair.

```bash
sudo cat privatekey | wg pubkey | sudo tee publickey
```

**Why:** Derives the public key from the private key. Public key is shared with the redirector. Private key stays here.

```bash
sudo chmod 600 privatekey publickey
```

**Why:** Only root can read these files. Stricter than the directory umask.

```bash
echo "=== PUBLIC KEY ===" && sudo cat /etc/wireguard/publickey
```

**Write down the public key.** The redirector needs it tomorrow to establish the tunnel.

### Create Config

```bash
sudo nano /etc/wireguard/wg0.conf
```

```ini
[Interface]
PrivateKey = <C2_PRIVATE_KEY>
Address = 10.0.1.1/24
ListenPort = 51820

PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT

[Peer]
# Redirector
PublicKey = <REDIRECTOR_PUBLIC_KEY>
AllowedIPs = 10.0.1.2/32
```

| Setting | Why |
|---------|-----|
| PrivateKey | This server's secret. Proves identity to the redirector. |
| Address 10.0.1.1/24 | Tunnel IP for this server. /24 means the tunnel network spans .1 to .254. |
| ListenPort 51820 | UDP port WireGuard listens on. Redirector sends encrypted packets here. |
| PostUp/PostDown | Allow traffic to flow through the tunnel interface when it starts/stops. |
| PublicKey (Peer) | Redirector's public key. Filled in tomorrow. Until then it is a placeholder. |
| AllowedIPs 10.0.1.2/32 | Only accept tunnel traffic from the redirector's tunnel IP. Drop everything else. |

Replace `<C2_PRIVATE_KEY>` with your actual private key. Leave `<REDIRECTOR_PUBLIC_KEY>` as is.

### Enable IP Forwarding

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

**Why:** Allows traffic to flow from the WireGuard tunnel interface to the Sliver listener. Without this packets arrive at the tunnel but go nowhere.

```bash
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-wireguard.conf
sudo sysctl --system
```

**Why:** Makes the setting permanent across reboots. First command is immediate. Second writes it to disk.

### Do NOT Start the Tunnel Yet

**Why:** The redirector is not configured. WireGuard needs both peers to have each other's keys before the tunnel can establish. Starting now would error.

Tomorrow after setting up the redirector:

```bash
sudo wg-quick up wg0
sudo systemctl enable wg-quick@wg0
```

---

## Step 9: Start HTTPS Listener

**Why:** This is where implants connect. We bind it to the WireGuard tunnel IP so implants can only reach it through the redirector's tunnel. Nobody on the internet can reach this listener directly.

```bash
sliver
```

```
sliver > https --lhost 10.0.1.1 --lport 8443
```

| Flag | Why |
|------|-----|
| --lhost 10.0.1.1 | Bind ONLY to the WireGuard tunnel interface IP. Not the public IP. Not the private IP. Only reachable through the tunnel. |
| --lport 8443 | Listen on port 8443. The redirector receives traffic on 443 and forwards it here through the tunnel. We use 8443 because 443 would conflict if we ran a web server. |

Expected output:

```
[*] Starting HTTPS listener on 10.0.1.1:8443 ...
[!] Self-signed certificates are detected by network scanners
[*] HTTPS listener started successfully
```

**Why self-signed cert warning is OK:** The redirector terminates TLS for us with a real Let's Encrypt certificate. Traffic between redirector and this listener is already encrypted by the WireGuard tunnel. It does not need its own TLS layer.

```
sliver > jobs
```

```
 ID   Name   Protocol   Bind Address
==== ====== ========== ===============
  1   https  tcp        10.0.1.1:8443
```

---

## Step 10: Enable Multi-Operator

**Why:** Multiple operators work the engagement simultaneously. One does discovery, another does lateral movement. Each gets their own certificate. Every command is logged with operator name for the final report.

```
sliver > multiplayer
```

```
[*] Multi-player mode enabled!
[*] Operators can connect to 10.0.1.1:31337
```

```
sliver > new-operator --name operator1 --lhost 10.0.1.1 --save /opt/sliver/operator1.cfg
sliver > new-operator --name yourname --lhost 10.0.1.1 --save /opt/sliver/yourname.cfg
```

| Flag | Why |
|------|-----|
| --name | Display name. All commands from this operator are logged with this name. |
| --lhost 10.0.1.1 | Tunnel IP operators connect to. They must be on the WireGuard network to connect. |
| --save /opt/sliver/... | File path for the operator config. Contains certificate + connection info. |

```
sliver > operators
sliver > exit
```

Config files at `/opt/sliver/*.cfg`. Distribute to operators after WireGuard tunnel is up tomorrow.

---

## Verify Everything

```bash
sudo systemctl status sliver        # Must show active (running)
sudo ufw status verbose             # deny incoming, SSH from 10.0.0.0/16, 51820/udp
sudo cat /etc/wireguard/wg0.conf    # Keys in place, peer is placeholder
sliver
sliver > jobs                        # HTTPS listener on 10.0.1.1:8443
sliver > exit
```

---

## What We Built Today

- Resource Group `rg-c2-infra` - delete this when engagement ends, everything disappears
- VNet `vnet-c2-infra` with subnet `default` (10.0.0.0/24) - private network for our VMs
- VM `c2-server` - Ubuntu 22.04, B1s (~$7/month), public IP locked down by NSG
- NSG blocks all inbound from internet - SSH only from inside VNet
- UFW blocks all inbound at OS level - second firewall layer
- Fail2ban bans IPs after 5 SSH failures - brute force protection
- Sliver running as systemd service - auto-starts on boot, auto-restarts on crash
- HTTPS listener on `10.0.1.1:8443` - only reachable through WireGuard tunnel
- Multi-operator gRPC on `10.0.1.1:31337` - team connects through tunnel
- WireGuard keys generated, config written - ready for redirector tomorrow

---

## Troubleshooting

**VM cannot download packages (apt/wget fail):**

- Portal → VNet `vnet-c2-infra` → Subnets → `default` → check outbound access is enabled
- Portal → VM → Networking → check public IP is attached and active

**Why this happens:** The 2026 Azure change. Private subnet blocks outbound. Either enable outbound on the subnet or confirm the public IP is attached.

**Sliver service fails to start:**

```bash
sudo journalctl -u sliver --no-pager -n 50
```

Common causes: binary not executable (`chmod +x`), `/opt/sliver` missing, unpack not run (`sliver-server unpack --force`).

**Locked out by UFW misconfiguration:**

Portal → VM → Connect → Serial console → login directly → `sudo ufw disable` → fix rules → `sudo ufw --force enable`

**Why Serial Console works:** Connects to the VM's virtual serial port. Bypasses all network firewalls. Always available.

**WireGuard fails tomorrow:**

```bash
sudo nano /etc/wireguard/wg0.conf
```

- PrivateKey must be exactly 44 base64 characters. No spaces. No line breaks inside the key.
- Address must have `/24` suffix.
- Peer PublicKey must be the redirector's actual public key (not the placeholder).

---

## Next Step

Continue to: [02_domain_redirector.md](02_domain_redirector.md)
