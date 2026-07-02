# Day 2: Setting Up Your Sliver C2 Server on Azure

---

## Where We Left Off

Yesterday you learned what C2 is, why we use Sliver, and why the redirector architecture is non-negotiable. Today we build it. For real.

By the end of today you will have:

1. Two Azure VMs running (C2 server + redirector)
2. Both locked down with firewalls at two layers (Azure NSG + UFW on the VM itself)
3. Sliver installed on the C2 server and running as a system service
4. An HTTPS listener active and verified working
5. Your operator console connected and ready

There are a few real problems we are going to hit along the way. Azure changed how networking works in early 2026, and the portal UI changed too. Old tutorials will not match what you see. I am going to walk you through exactly what the current Azure portal looks like and how to deal with everything that trips people up.

Let me be direct: this is not a short setup. Do not rush it. Every step matters. If you skip something or eyeball a config, you will spend hours debugging. Take your time. Do each step, verify it works, then move on.

---

## What We Are Building Today

Two Azure VMs, both Ubuntu 22.04 LTS:

| VM | Purpose | Open Ports | Internet Facing? |
|----|---------|-----------|-----------------|
| `c2-server` | Runs Sliver C2. Nobody talks to this directly except us (SSH) and the redirector (port 8443) | 22 (SSH, your IP only), 8443 (redirector IP only) | No |
| `redirector` | Runs Nginx. Receives implant traffic, forwards to C2 server | 443 (public), 22 (SSH, your IP only) | Yes (port 443 only) |

One shared Azure Virtual Network (VNet) with one subnet. Both VMs sit inside the same private network. The redirector's port 443 is exposed to the internet. The C2 server is not directly reachable from the internet at all except for your SSH.

---

## Before You Touch Azure: Know Your Own Public IP

Before you create anything, find out your current public IP address. You need this for the firewall rules. Your SSH access and your NSG rules will be locked to this IP.

Go to this URL in your browser:

```
https://ifconfig.me
```

Write down the IP it shows. For this guide, we will call it `YOUR_OPERATOR_IP`. Replace `YOUR_OPERATOR_IP` with your actual IP everywhere you see it.

If you are on a VPN or your IP changes frequently, you will need to update the NSG rules every time your IP changes. That is why we say: do engagement work from a dedicated VM or a fixed location.

---

## CRITICAL: The Azure 2026 Private Subnet Change

**Read this before you create anything. If you skip this section, your VMs will have no internet access and you will spend hours wondering why `apt update` fails.**

### What Changed

On March 31, 2026, Microsoft changed how new Azure Virtual Networks work. Before this date, when you created a VM in a new VNet, Azure would silently assign it a temporary public IP for outbound traffic even if you did not ask for one. This was called Default Outbound Access. Your VM could reach the internet (for `apt`, GitHub, etc.) without you doing anything special.

That is gone now.

After March 31, 2026, any new VNet you create uses what Azure calls **private subnets by default**. This means VMs in that VNet have **zero outbound internet access** by default. When you try to run `apt update` after setting up your VM, it will hang and then fail. When the Sliver installer tries to pull packages from GitHub, it will fail. Nothing will work until you explicitly configure outbound internet access.

This trips everyone up the first time. You are not doing anything wrong. Azure just changed the rules.

### What You Will See When It Happens

You SSH into your new VM and run:

```bash
sudo apt update
```

You will see something like:

```
Err:1 http://azure.archive.ubuntu.com/ubuntu jammy InRelease
  Could not connect to azure.archive.ubuntu.com:80
Err:2 http://security.ubuntu.com/ubuntu jammy-security InRelease
  Could not connect to security.ubuntu.com:80
Reading package lists... Done
W: Failed to fetch http://azure.archive.ubuntu.com/ubuntu/dists/jammy/InRelease
```

Or it just hangs with a blinking cursor for 30 seconds then times out. That is the private subnet blocking all outbound traffic.

### The Fix: Azure NAT Gateway

A NAT Gateway is an Azure resource that gives your private subnet a fixed, stable outbound internet connection. All VMs in the subnet go through the NAT Gateway when they need to reach the internet. From the internet's perspective, those VMs all share the NAT Gateway's public IP.

This is actually good for us from an OPSEC standpoint too. Both our VMs (C2 server and redirector) need internet access during setup. After setup, only the redirector needs to stay internet-facing on port 443. The C2 server just needs outbound access for package updates.

**Here is the order of operations we follow to avoid all the problems:**

1. Create the Resource Group
2. Create the Virtual Network and Subnet (standalone, NOT inline during VM creation)
3. Create the NAT Gateway and associate it with the subnet
4. THEN create both VMs, picking our pre-made VNet from dropdowns

This order matters. The Azure portal VM creation wizard has changed in 2026, and trying to set up networking inline during VM creation causes confusion. We do networking first, then VMs.

---

## Step 1: Create the Resource Group

A Resource Group is just a container. Everything for this engagement goes in one Resource Group so you can delete everything cleanly when the engagement ends.

1. Go to [portal.azure.com](https://portal.azure.com) and sign in
2. In the top search bar, type **Resource groups** and click it
3. Click **+ Create**
4. Fill in:
   - **Subscription:** your subscription
   - **Resource group:** `c2-infra-rg` (or whatever name you want, just be consistent)
   - **Region:** pick a region. Use `East US` or `West Europe`. Pick one and stick with it. All resources must be in the same region.
5. Click **Review + create**, then **Create**

You will see the Resource Group appear in your list in a few seconds.

---

## Step 2: Create the Virtual Network and Subnet (Do This BEFORE VMs)

**Why we do this before the VMs:** The new Azure portal VM creation wizard does not give you fine-grained control over subnet settings inline. If you try to create the VNet during VM creation, you get a simplified wizard that may not show all the options. By creating the VNet first as a standalone resource, you have full control. Then during VM creation, you just pick it from a dropdown.

### Create the VNet

1. In the search bar, type **Virtual networks** and click it
2. Click **+ Create**
3. **Basics tab:**
   - **Resource group:** `c2-infra-rg`
   - **Name:** `c2-vnet`
   - **Region:** same region you chose above
4. **Address space tab:**
   - Azure will suggest a virtual network address space of `10.0.0.0/16`. That is fine. Leave it.
   - Under the **Subnets** list, you will see a subnet named `default` with address range `10.0.0.0 - 10.0.0.255`.
   - Click the pencil (edit) icon on the right side of the `default` subnet row. This opens the **Edit subnet** side panel.
   - Configure the following fields in the panel:
     - **Name:** Change from `default` to `c2-subnet`.
     - **IPv4 address range:** This defaults to the parent VNet space `10.0.0.0/16`. Keep it as is.
     - **Starting address:** `10.0.0.0`.
     - **Size:** Select `/24 (256 addresses)` from the dropdown. This automatically sets the subnet address range to `10.0.0.0 - 10.0.0.255`.
     - **Enable private subnet (no default outbound access):** Make sure this is **checked**. This is the March 31, 2026 default security behavior we discussed. It forces the subnet to be private.
     - **NAT gateway:** Leave as `None` (we will create and link this in Step 3).
     - **Network security group:** Leave as `None`.
   - Click **Save** at the bottom of the panel to update the subnet.

   > **Note on Private Subnets (March 2026 Change):** By leaving "Enable private subnet" checked, Azure will block all implicit outbound internet access from this subnet. This is why our VMs will not be able to download packages initially. We will explicitly fix this in Step 3 by creating a NAT Gateway and attaching it to this subnet.

5. **Security tab:** Skip, nothing to change here
6. Click **Review + create**, then **Create**

Wait for the VNet to deploy. It takes about 30 seconds.

---

## Step 3: Create the NAT Gateway (Fixes the Internet Problem)

Now we give our subnet outbound internet access. This must be done before the VMs are created, because the Sliver install script needs to download from GitHub.

1. In the search bar, type **NAT gateways** and click it.
2. Click **+ Create**.
3. **Basics tab:**
   - **Resource group:** `c2-infra-rg`
   - **NAT gateway name:** `c2-nat-gateway`
   - **Region:** Select the same region as your virtual network.
   - **SKU:** Leave it on the default **Standard V2 (Recommended)** (or select **Standard**).
   - **Enable NAT64:** Leave unchecked.
   - **TCP idle timeout (minutes):** `4`.
4. Click **Next: Outbound IP**.
5. **Outbound IP tab:**
   - Click **+ Add public IP addresses or prefixes**. This opens the **Manage public IP addresses and prefixes** side panel.
   - Inside this panel, click the blue **Create a public IP address** link (located right below the dropdown).
   - In the nested **Add a public IP address** overlay, configure:
     - **Name:** `c2-nat-pip`.
     - **IP version:** Select **IPv4**.
     - **SKU:** This defaults to **StandardV2** and will be greyed out/locked to match your NAT Gateway Basics tab selection.
     - **Assignment:** Static (locked default).
   - Click **OK** on the overlay, then click **OK** (or close) on the Manage side panel.
   - You will now see `c2-nat-pip` added to the **Public IP addresses** table.
6. Click **Next: Networking**.
7. **Networking tab:**
   - Under **Virtual network**, select `c2-vnet` from the dropdown list.
   - Under **Select specific subnets**, select `c2-subnet` (check the box next to it).
8. Click **Review + create**, then **Create**.

Deployment takes about 1-2 minutes.

**Verify it worked:** Once deployed, go to your `c2-vnet` resource, click **Subnets**, and click on `c2-subnet`. You should see **NAT gateway: c2-nat-gateway** listed. That means your subnet now has outbound internet access.

---

## Step 4: Create the Network Security Groups

NSGs are Azure's firewall. They sit in front of the VM's network interface and block or allow traffic based on rules you define. We create two NSGs, one per VM, with different rules.

Think of it this way: the NSG is the first checkpoint. Traffic hits the NSG before it even reaches the VM. If the NSG says no, the VM never sees the packet. Then inside the VM, UFW is a second checkpoint. Two layers. If you misconfigure one, the other catches it.

### NSG for the C2 Server

This VM should not be reachable from the internet except for your SSH. The only other thing that should reach it is the redirector (on port 8443 for C2 traffic). We do not know the redirector's IP yet (we have not created it), so we will add that rule after we create the redirector VM.

1. In the search bar, type **Network security groups** and click it
2. Click **+ Create**
3. Fill in:
   - **Resource group:** `c2-infra-rg`
   - **Name:** `c2-server-nsg`
   - **Region:** same region
4. Click **Review + create**, then **Create**
5. Once created, go to the NSG resource. In the left menu, find the **Settings** section and click on **Inbound security rules**.
6. Click **+ Add** (at the top of the rules list) and create this rule:

   | Field | Value |
   |-------|-------|
   | Source | IP Addresses |
   | Source IP addresses | `YOUR_OPERATOR_IP` |
   | Source port ranges | `*` |
   | Destination | Any |
   | Service | SSH |
   | Destination port ranges | `22` |
   | Protocol | TCP |
   | Action | Allow |
   | Priority | `100` |
   | Name | `Allow-SSH-OperatorOnly` |

7. Click **Add**

That is the only inbound rule you add now. The NSG already has a default `DenyAllInBound` rule at priority 65500. Everything not explicitly allowed is blocked. We will add the rule for the redirector later.

**Check your outbound rules:** Click **Outbound security rules**. You should see an `AllowInternetOutBound` rule at priority 65001. This allows your VM to make outbound connections (needed for Sliver to reach the implant via the redirector, and for package downloads via NAT gateway). Leave this as is.

### NSG for the Redirector

This VM needs port 443 open to the public internet so that implants can connect, plus port 22 open only to your operator IP for SSH configuration.

1. In the search bar, type **Network security groups** and click it.
2. Click **+ Create**.
3. Fill in the basics:
   - **Resource group:** `c2-infra-rg` (select from dropdown)
   - **Name:** `redirector-nsg`
   - **Region:** Select the same region as before.
4. Click **Review + create**, then click **Create**.
5. Once deployment completes, navigate to the `redirector-nsg` resource.
6. In the left menu, go to the **Settings** section and click on **Inbound security rules**.
7. Now add the two rules by clicking **+ Add** at the top of the rules list:

   **Rule 1: SSH access for you only**
   This locks down administrative SSH access to your specific operator IP.

   | Form Field | Value | Explanation |
   |------------|-------|-------------|
   | **Source** | `IP Addresses` | We are filtering traffic by a specific IP source. |
   | **Source IP addresses** | `YOUR_OPERATOR_IP` | Replace this with your exact IP (e.g., `157.50.74.229`). Azure automatically treats a single IP as a `/32` single-host entry. |
   | **Source port ranges** | `*` | The client port can be any random high port. |
   | **Destination** | `Any` | Directs to any interface on the VM. |
   | **Service** | `SSH` | Automatically sets the protocol and port. |
   | **Destination port ranges** | `22` | The port SSH listens on. |
   | **Protocol** | `TCP` | SSH runs over TCP. |
   | **Action** | `Allow` | We want to let this traffic pass. |
   | **Priority** | `100` | Process this first. |
   | **Name** | `Allow-SSH-OperatorOnly` | Descriptive name to track the rule. |

   Click **Add**. Wait for the rule to save.

   **Rule 2: HTTPS public access for incoming C2 traffic**
   This must be open to everyone (`Any`) because we do not know the public IP addresses of the compromised machines beforehand.

   | Form Field | Value | Explanation |
   |------------|-------|-------------|
   | **Source** | `Any` | Anyone on the internet can connect. |
   | **Source port ranges** | `*` | Any source port. |
   | **Destination** | `Any` | Directs to any interface on the VM. |
   | **Service** | `Custom` | We will specify port 443 manually. |
   | **Destination port ranges** | `443` | The standard port for HTTPS/TLS traffic. |
   | **Protocol** | `TCP` | HTTPS runs over TCP. |
   | **Action** | `Allow` | Let the traffic pass. |
   | **Priority** | `110` | Process this after the SSH rule. |
   | **Name** | `Allow-HTTPS-Public` | Descriptive name. |

   Click **Add** and wait for the rule to save.

Both rules are now active. Any packet trying to hit the redirector VM will be dropped by the default `DenyAllInBound` rule (priority 65500) unless it matches one of these two rules.

---

## Step 5: Create the C2 Server VM

Now we create the first VM. Because we already have the VNet and NSG ready, the VM creation is clean.

1. In the search bar, type **Virtual machines** and click it
2. Click **+ Create** > **Azure virtual machine**

### Basics Tab

| Field | Value |
|-------|-------|
| Resource group | `c2-infra-rg` |
| Virtual machine name | `c2-server` |
| Region | same region |
| Availability options | No infrastructure redundancy required |
| Image | **Ubuntu Server 22.04 LTS** |
| Size | **Standard_B2s** (2 vCPU, 4 GiB RAM) |
| Authentication type | **Password** |
| Username | `c2admin` |
| Password | Create a strong password. Write it down. You need it to SSH in. |
| Confirm password | same |
| Public inbound ports | **None** (we handle this in the NSG) |

**A note on password vs SSH keys:**

You might read guides that say SSH keys only, no passwords. That is the right approach for production servers. For this engagement VM setup, we use a password for simplicity because:

- Both VMs sit behind NSGs that only allow SSH from your specific operator IP
- Nobody on the internet can even reach port 22 on these VMs (NSG blocks it)
- We add fail2ban in aggressive mode inside the VM as a second layer
- Azure Serial Console gives you a backup way to access the VM if you mess up SSH settings

The real risk with passwords is brute force. But if only your IP can reach port 22, there is nobody to brute force from. The NSG handles that at the network layer before the connection ever gets to the VM.

We still install fail2ban because NSG rules can have mistakes, and defense in depth is always the right call.

### Disks Tab

Leave defaults. Standard SSD OS disk is fine for this.

### Networking Tab

Configure the network settings for the VM:

1. **Virtual network:** Select `c2-vnet` from the dropdown list.
2. **Subnet:** Select `c2-subnet`.
3. **Public IP:** Click **Create new** right below the Public IP field. This opens the **Create public IP address** side overlay.
   - **Name\*:** `c2-server-pip`.
   - **SKU:** Defaults to `Standard` (read-only).
   - **IP version:** `IPv4`.
   - **IP address assignment:** `Static`.
   - **Tier:** `Regional`.
   - **Availability zone\*:** Select any zone available in the dropdown (e.g., `1`, `2`, `3`, or `No zone`).
   - Click **OK** at the bottom of the overlay to save the public IP settings.
4. **NIC network security group:** Select **Advanced** (or **Configure**).
5. **Configure network security group:** Select the existing security group **c2-server-nsg**.

> **2026 UI Note:** If you do not see an "Advanced" option for NSG, look for a "Configure" link or a dropdown that says "Basic". Click on it. In the current portal version, the NSG selector may be under a "Configure" expandable section. If you cannot find it at all, finish creating the VM, then go to the VM's Network Interface resource after creation and assign the NSG there. The NSG can be attached to a NIC at any time, not just during VM creation.

**Public IP:** Yes, the C2 server gets a public IP, but only for your SSH access. We will firewall it so only your operator IP can reach port 22. No other ports are accessible.

### Monitoring Tab (or Management Tab in older UI)

Click on the **Monitoring** tab at the top. Scroll down to find **Boot diagnostics**:
- In the new Azure portal interface, Boot Diagnostics is enabled by default with a managed storage account.
- Ensure **Boot diagnostics** is set to **Enable with managed storage account** (or **Enabled**). 
- This is what enables the **Azure Serial Console**, giving you a backup browser-based terminal if you ever accidentally lock yourself out of SSH via UFW or NSG misconfiguration.

### Review + Create

Click through, confirm settings look correct, click **Create**. Deployment takes 2-4 minutes.

**Once deployed:** Note the public IP of this VM. Write it down. This is your `C2_SERVER_IP`. You need it to SSH in, and later you need it to configure Nginx on the redirector to forward traffic here.

---

## Step 6: Create the Redirector VM

Same process. Create a second VM.

| Field | Value |
|-------|-------|
| Resource group | `c2-infra-rg` |
| Virtual machine name | `redirector` |
| Region | same region |
| Image | **Ubuntu Server 22.04 LTS** |
| Size | **Standard_B1s** (1 vCPU, 1 GiB RAM, cheap) |
| Authentication type | **Password** |
| Username | `c2admin` |
| Password | same password as C2 server (your call, just write it down) |
| Public inbound ports | **None** |

Networking Tab:

1. **Virtual network:** Select `c2-vnet`.
2. **Subnet:** Select `c2-subnet`.
3. **Public IP:** Click **Create new**:
   - **Name\*:** `redirector-pip`.
   - **Availability zone\*:** Select any zone (e.g., `1`, `2`, `3`, or `No zone`).
   - Click **OK**.
4. **NIC network security group:** Select **Advanced** (or **Configure**) -> select `redirector-nsg`.

Monitoring Tab: Ensure Boot diagnostics is enabled (same as C2 server).

Click **Review + create**, then **Create**.

Once deployed, note this VM's public IP. This is your `REDIRECTOR_IP`.

---

## Step 7: Add the Redirector Rule to the C2 Server NSG

Now that you have both IPs, go back to `c2-server-nsg` and add the rule that allows the redirector to forward C2 traffic to the C2 server.

1. Go to **Network security groups** > **c2-server-nsg** > **Inbound security rules**
2. Click **+ Add**

| Field | Value |
|-------|-------|
| Source | IP Addresses |
| Source IP | `REDIRECTOR_IP/32` |
| Destination port ranges | `8443` |
| Protocol | TCP |
| Action | Allow |
| Priority | `110` |
| Name | `Allow-C2-FromRedirector` |

Click **Add**.

Now the C2 server accepts:
- SSH on port 22 from your operator IP
- TCP on port 8443 from the redirector only
- Everything else: blocked

---

## Step 8: SSH Into Both VMs and Do Initial Hardening

You are going to open two terminal windows, one for each VM. Do not close them. You will be switching back and forth.

### Connect to Both VMs

On your operator machine, open a terminal and SSH in:

```bash
# C2 Server
ssh c2admin@C2_SERVER_IP

# Redirector (open a second terminal tab)
ssh c2admin@REDIRECTOR_IP
```

When prompted about the host fingerprint, type `yes`. Enter the password you set.

If SSH refuses to connect, check:
1. Did you save your NSG rules? (Go back to Azure portal and confirm)
2. Is your current IP the same one you put in the NSG rule? Check with `https://ifconfig.me` again.
3. Did the VM finish deploying? (Go to the VM overview page and check status)

### Verify Outbound Internet Works (First Test After the 2026 Change)

On the C2 server, immediately run:

```bash
ping -c 4 8.8.8.8
```

You should see replies. If you see `Network is unreachable` or `100% packet loss`:

```
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
From 10.0.0.4 icmp_seq=1 Destination Net Unreachable
```

Your NAT Gateway is not associated with the subnet correctly. Go back to the Azure portal:
- Find the `c2-nat-gateway` resource
- Click **Subnets**
- Check that `c2-vnet/c2-subnet` is listed there
- If not, click **+ Associate** and add it

Once you see ping replies, continue.

### Update the System

Run this on BOTH VMs:

```bash
sudo apt update && sudo apt upgrade -y
```

You will see the package list download, then package upgrades. This takes 2-5 minutes. Wait for it to finish completely.

If this hangs: NAT Gateway issue. Fix it as described above.

```
You will see output like:
Get:1 http://azure.archive.ubuntu.com/ubuntu jammy InRelease [270 kB]
Get:2 http://azure.archive.ubuntu.com/ubuntu jammy-updates InRelease [128 kB]
...
Fetched 28.2 MB in 12s (2,350 kB/s)
Reading package lists... Done
```

That output means internet is working and packages are downloading. Good.

---

## Step 9: Install fail2ban on Both VMs

Before we do anything else, we add the brute force protection layer. This goes on both VMs.

Run on both:

```bash
sudo apt install -y fail2ban
```

Now configure it. Create a local config file (never edit jail.conf directly, it gets overwritten on updates):

```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```

Find the `[DEFAULT]` section near the top. Update these values:

```ini
[DEFAULT]
# Whitelist your operator IP so you cannot ban yourself
ignoreip = 127.0.0.1/8 ::1 YOUR_OPERATOR_IP

# Ban for 24 hours
bantime = 86400

# Window to count failures: 10 minutes
findtime = 600

# Max failures before ban
maxretry = 3
```

Now find the `[sshd]` section and set it to aggressive mode:

```ini
[sshd]
enabled = true
mode = aggressive
port = ssh
logpath = %(sshd_log)s
backend = %(sshd_backend)s
```

Save the file (Ctrl+O, Enter, Ctrl+X in nano).

Apply the config:

```bash
# Validate no syntax errors
sudo fail2ban-client -t

# You should see: OK: configuration test is successful

# Restart fail2ban
sudo systemctl restart fail2ban

# Verify it is running
sudo systemctl status fail2ban
```

You should see `active (running)` in the status output.

Check the SSH jail is active:

```bash
sudo fail2ban-client status sshd
```

Expected output:

```
Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  |- Total failed: 0
|  `- Journal matches:  _SYSTEMD_UNIT=ssh.service + _COMM=sshd
`- Actions
   |- Currently banned: 0
   |- Total banned: 0
   `- Banned IP list:
```

Zero failures, zero bans. Good. fail2ban is watching.

**What fail2ban does:** If any IP (including yours if you are not in ignoreip) fails SSH login 3 times within 10 minutes, it gets banned for 24 hours. The ban means iptables drops all connections from that IP. Nobody brute forces your SSH.

**The aggressive mode** adds extra filters for SSH-specific attack patterns beyond just failed passwords: it catches invalid user attempts, pre-authentication disconnects, and specific DDoS patterns.

---

## Step 10: Disable the Cloud-Init SSH Override (Important Ubuntu-Azure Gotcha)

On Ubuntu VMs in Azure, there is a cloud-init file that can override your SSH settings. If you ever try to harden SSH later, this file will fight you and your changes will not stick after a reboot.

Fix it now on both VMs:

```bash
# Check if this file exists
ls /etc/ssh/sshd_config.d/

# You will likely see: 50-cloud-init.conf
# This file might force PasswordAuthentication yes or other settings
# Since we want passwords anyway, let us just check what it says

cat /etc/ssh/sshd_config.d/50-cloud-init.conf
```

You will see something like:

```
PasswordAuthentication yes
```

For our setup this is fine since we want password auth. But note this for the future: if you ever want to switch to key-only auth on an Azure Ubuntu VM, you need to edit or remove this file first, THEN edit sshd_config. Otherwise your sshd_config changes get ignored.

---

## Step 11: Install Sliver on the C2 Server

Now we install Sliver. This all happens on the **C2 server VM**, not the redirector.

Make sure you are in the C2 server SSH session.

### Run the Sliver Install Script

```bash
curl https://sliver.sh/install | sudo bash
```

This script does several things:
1. Downloads the Sliver server binary
2. Downloads `mingw-w64` (cross-compiler needed to build Windows implants)
3. Installs Sliver as a systemd service
4. Creates the default config directories

The output will look like this:

```
[*] Downloading sliver-server...
[*] Installing mingw-w64 cross-compiler...
[*] Installing dependencies...
[*] Creating systemd service...
[*] Sliver installed to /usr/local/bin/sliver-server
```

This takes 3-5 minutes depending on connection speed. Wait for it.

### Start and Enable the Sliver Service

```bash
# Start the service
sudo systemctl start sliver

# Enable it to start on boot (the install script does NOT do this by default)
sudo systemctl enable sliver

# Check it is running
sudo systemctl status sliver
```

Expected output:

```
● sliver.service - Sliver
     Loaded: loaded (/etc/systemd/system/sliver.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2026-07-02 10:30:00 UTC; 5s ago
   Main PID: 1234 (sliver-server)
```

`active (running)` is what you want. `enabled` means it will start on reboot.

### Open the Sliver Console

```bash
sudo sliver-server
```

The first time it runs, it generates cryptographic keys, creates config files, and shows the Sliver banner:

```

          ██████  ██▓     ██▓ ██▒   █▓▓█████  ██▀███
        ▒██    ▒ ▓██▒    ▓██▒▓██░   █▒▓█   ▀ ▓██ ▒ ██▒
        ░ ▓██▄   ▒██░    ▒██▒ ▓██  █▒░▒███   ▓██ ░▄█ ▒
          ▒   ██▒▒██░    ░██░  ▒██ █░░▒▓█  ▄ ▒██▀▀█▄
        ▒██████▒▒░██████▒░██░   ▒▀█░  ░▒████▒░██▓ ▒██▒
        ▒ ▒▓▒ ▒ ░░ ▒░▓  ░░▓    ░ ▐░  ░░ ▒░ ░░ ▒▓ ░▒▓░
        ...

        All hackers gain OPERATOR
        https://github.com/BishopFox/sliver


[*] Server v1.5.x - Bishop Fox

sliver >
```

You are now in the Sliver console. The prompt is `sliver >`.

---

## Step 12: Create the HTTPS Listener

This is where Sliver starts listening for implant connections. We are setting up the listener on port 8443 (not 443) because Nginx on the redirector handles 443. Nginx forwards traffic from port 443 to Sliver on port 8443.

In the Sliver console, type:

```
sliver > https --lport 8443
```

Output you will see:

```
[*] Starting HTTPS :8443 listener ...
[!] Successfully started job #1

sliver >
```

Verify the listener is active:

```
sliver > jobs
```

Output:

```
 ID   Name    Protocol   Port   Domains
==== ======== ========== ====== =========
 1   https    tcp        8443
```

Job ID 1, protocol HTTPS, port 8443. That is your listener running.

### Why Port 8443 and Not 443?

Port 443 on the C2 server is not open to the internet. Only port 8443 is accessible, and only from the redirector's IP (per the NSG rule we set). Sliver does not need to handle TLS certificates because Nginx handles TLS termination at the redirector. Sliver just receives the forwarded HTTP traffic from Nginx on port 8443.

This means Sliver's port 8443 uses its own built-in encryption (Sliver's implant communication is encrypted with the per-implant asymmetric keys regardless), but the external HTTPS that the implant actually uses is handled by Nginx and the Let's Encrypt certificate on the redirector.

---

## Step 13: Test the Listener Locally

Before we configure the redirector, verify that Sliver is actually listening and responding on port 8443 on the C2 server.

Open a second terminal (or in the same terminal, use Ctrl+Z to background Sliver temporarily, then `fg` to bring it back).

Better: open a second SSH session to the C2 server and test from there:

```bash
# In a second SSH session to the C2 server
curl -kv https://127.0.0.1:8443
```

The `-k` flag skips certificate verification (Sliver uses a self-signed cert internally, which is fine since Nginx handles the real cert externally).

You will see a lot of TLS handshake output:

```
* Trying 127.0.0.1:8443...
* Connected to 127.0.0.1 (127.0.0.1) port 8443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
...
< HTTP/1.1 404 Not Found
```

A `404 Not Found` response is CORRECT and expected. Sliver is running. It says 404 because you are not sending a proper implant request, just a raw curl. The implant would send a specially formatted HTTPS request that Sliver knows how to parse. Raw curl gets a 404. That is Sliver telling you "I am here but I do not know what you want."

**If you get `Connection refused`:** The listener is not running. Go back to the Sliver console and run `https --lport 8443` again.

**If you get `curl: (7) Failed to connect`:** Check that nothing else is on port 8443. Run `sudo ss -tlnp | grep 8443` to see what is listening on that port.

---

## Step 14: Open the Multiplayer Port (Connect From Your Operator Machine)

Right now you can only interact with Sliver by SSHing into the server and running `sudo sliver-server`. That is fine for solo work, but the proper setup lets you run a Sliver client on your operator machine and connect to the server remotely over an encrypted mTLS connection.

This is the multiplayer mode. You need it for team engagements.

In the Sliver console, enable multiplayer:

```
sliver > multiplayer
```

Output:

```
[*] Multiplayer mode enabled!
```

This starts a listener on port 31337 (TCP). The Sliver operator client on your local machine will connect to this port using an mTLS certificate that the server generates for you.

Now create an operator config for yourself:

```
sliver > new-operator --name yourname --lhost C2_SERVER_IP
```

Replace `yourname` with your name (no spaces) and `C2_SERVER_IP` with the actual public IP of your C2 server.

Output will be something like:

```
[*] Generating new client certificate, please wait...
[*] Saved new client config to: /root/yourname_C2_SERVER_IP.cfg
```

This `.cfg` file contains the certificate and connection details for your operator client. You need to copy it to your operator machine.

**Copy the config file to your operator machine:**

In a terminal on your operator machine (not the SSH session), run:

```bash
scp c2admin@C2_SERVER_IP:/root/yourname_C2_SERVER_IP.cfg ~/sliver-client.cfg
```

Enter the VM password when prompted.

Now you need the Sliver client binary on your local machine. Download it:

```bash
# On your operator machine (Linux/Mac)
curl -L https://github.com/BishopFox/sliver/releases/latest/download/sliver-client_linux -o sliver-client
chmod +x sliver-client

# Import your operator config
./sliver-client import ~/sliver-client.cfg

# Connect to the server
./sliver-client
```

You should see the Sliver banner and be connected to your remote server.

**For now, the SSH console method is fine.** Setting up the client on your operator machine is optional for solo work. The SSH method works perfectly well.

**OPSEC note on port 31337:** This multiplayer port should also be locked down in your NSG. Currently our NSG only allows port 22 (SSH) and port 8443 (from redirector) on the C2 server. Port 31337 is already blocked from the internet by the NSG default deny rule. If you want to use the remote client, add a rule allowing port 31337 from your operator IP only, similar to the SSH rule.

---

## Step 15: UFW Firewall on the C2 Server (Second Layer)

NSG is Azure's firewall. UFW is the Linux VM's own firewall. We want both because if someone misconfigures the NSG, UFW still protects the VM.

On the C2 server:

```bash
# Set defaults: deny incoming, allow outgoing
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH from your operator IP only
sudo ufw allow from YOUR_OPERATOR_IP to any port 22 proto tcp

# Allow C2 traffic from the redirector IP only
sudo ufw allow from REDIRECTOR_IP to any port 8443 proto tcp

# Enable UFW
# WARNING: make sure you added your SSH rule before enabling, or you lock yourself out
sudo ufw enable
```

When you run `sudo ufw enable`, it will warn you:

```
Command may disrupt existing ssh connections. Proceed with operation (y|n)?
```

Type `y`. Your current SSH session stays open. New connections from IPs not in the rules will be blocked.

Verify your rules:

```bash
sudo ufw status numbered
```

Output should look like:

```
Status: active

     To                         Action      From
     --                         ------      ----
[ 1] 22/tcp                     ALLOW IN    YOUR_OPERATOR_IP
[ 2] 8443/tcp                   ALLOW IN    REDIRECTOR_IP
```

Two rules. Everything else is denied by default. 

---

## Step 16: UFW Firewall on the Redirector VM

Switch to your redirector SSH session and set up UFW there too:

```bash
# Set defaults
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow your SSH
sudo ufw allow from YOUR_OPERATOR_IP to any port 22 proto tcp

# Allow HTTPS from anywhere (implant traffic)
sudo ufw allow 443/tcp

# Enable
sudo ufw enable
```

Verify:

```bash
sudo ufw status numbered
```

```
Status: active

     To                         Action      From
     --                         ------      ----
[ 1] 22/tcp                     ALLOW IN    YOUR_OPERATOR_IP
[ 2] 443/tcp                    ALLOW IN    Anywhere
[ 3] 443/tcp (v6)               ALLOW IN    Anywhere (v6)
```

Port 22 only from you. Port 443 from the world. Everything else blocked.

---

## Step 17: Verify Your Complete Firewall Stack

Run these tests from your operator machine (not inside the VMs):

**Test 1: Can you SSH into the C2 server?**
```bash
ssh c2admin@C2_SERVER_IP
```
Should connect. Enter password.

**Test 2: Can you NOT reach the C2 server's port 8443 from your operator machine?**
```bash
curl -kv --connect-timeout 5 https://C2_SERVER_IP:8443
```
Should time out. You are not the redirector, so this port is blocked from you at the NSG level.

**Test 3: Can you SSH into the redirector?**
```bash
ssh c2admin@REDIRECTOR_IP
```
Should connect.

**Test 4: Can you reach the redirector's port 443 from the internet?**
```bash
curl -kv --connect-timeout 5 https://REDIRECTOR_IP
```
This should connect (though you will get an error or 502/connection refused because Nginx is not installed yet). But it should at least connect, not time out. Port 443 is open.

If test 1 and 3 pass and test 2 times out, your firewall setup is correct.

---

## Troubleshooting: Common Issues

### Issue: `apt update` fails with connection errors

**Cause:** Azure 2026 private subnet change. Your subnet has no outbound internet.

**Fix:**
1. Go to Azure portal, find `c2-nat-gateway`
2. Click **Subnets**
3. If your subnet is not listed, click **+ Associate**, select `c2-vnet` and `c2-subnet`
4. Wait 1-2 minutes, try `apt update` again

### Issue: Sliver install fails halfway through with network errors

**Cause:** Same as above. The install script downloads from GitHub and the connection was cut.

**Fix:** Fix the NAT Gateway first (above), then run the install command again:
```bash
curl https://sliver.sh/install | sudo bash
```

The script is idempotent, running it again is safe.

### Issue: `sudo sliver-server` gives "permission denied" or "command not found"

**Cause:** The install did not complete properly, or you are running as the wrong user.

**Fix:**
```bash
# Check if the binary exists
ls -la /usr/local/bin/sliver-server

# If not there, re-run the install
curl https://sliver.sh/install | sudo bash

# Try running with sudo explicitly
sudo /usr/local/bin/sliver-server
```

### Issue: `https --lport 8443` says "address already in use"

**Cause:** Something else is on port 8443, or Sliver already has a listener there (from a previous session).

**Fix:**
```bash
# Check what is on 8443
sudo ss -tlnp | grep 8443

# Inside Sliver, list current listeners
sliver > jobs

# Kill a listener
sliver > jobs -k 1
```

Then start a new listener.

### Issue: You cannot SSH in anymore. You locked yourself out.

**Cause:** Possible mistake in UFW rules, or your operator IP changed.

**Fix:** Azure Serial Console (this is why we enabled Boot Diagnostics).

1. Go to Azure portal
2. Find your VM
3. Click **Help** > **Serial console** in the left menu
4. The console opens in your browser
5. Press Enter, you should see a login prompt
6. Log in with `c2admin` and your password
7. Now you are inside the VM without SSH
8. Run `sudo ufw status` to check rules
9. If your IP changed, update the rule:
   ```bash
   sudo ufw delete allow from OLD_IP to any port 22
   sudo ufw allow from NEW_IP to any port 22 proto tcp
   ```
10. Also update your NSG rule in the Azure portal

**If Serial Console is not available:** Go to VM > **Help** > **Reset password**. This lets you reset the password or SSH config through Azure's VM agent without network access.

### Issue: Azure portal VM creation does not show "Advanced" NSG option

**Cause:** The 2026 portal UI changed. The NSG selector was moved.

**Fix:**
1. Create the VM without selecting an NSG during creation (just leave it on Basic or None)
2. After the VM deploys, go to the VM resource
3. Click **Networking** in the left menu
4. Click on the Network Interface link
5. Click **Network security group** in the NIC settings
6. Click **Edit**, then select your existing NSG from the dropdown
7. Save

The NSG is applied to the NIC, not the VM directly. You can always attach or change it after creation.

---

## Status Check: Where We Are

By the end of this step, you should have:

| Item | Status |
|------|--------|
| Resource Group `c2-infra-rg` | Created |
| VNet `c2-vnet` with `c2-subnet` | Created |
| NAT Gateway attached to subnet | Working (apt update succeeds) |
| NSG `c2-server-nsg` | Created, SSH from your IP, 8443 from redirector |
| NSG `redirector-nsg` | Created, SSH from your IP, 443 from anywhere |
| `c2-server` VM | Running, fail2ban active, UFW active |
| `redirector` VM | Running, fail2ban active, UFW active |
| Sliver | Installed, systemd service running and enabled |
| HTTPS listener on 8443 | Active (verified with curl -kv, got 404) |

If all of those are green, you are ready to move to the redirector setup tomorrow. The Sliver side is complete. The C2 server is locked down. The next step is getting the redirector's Nginx configured with a real domain and a real SSL certificate, then we can test the full chain end to end.

---

## Saving Your State Before You Disconnect

Before you close your SSH sessions, do this inside the Sliver console:

```
sliver > jobs
```

Write down what you see. The job number and port. When you reconnect tomorrow, run:

```bash
sudo sliver-server
```

And verify the listener is still there:

```
sliver > jobs
```

If the listener is gone after a reboot, restart it:

```
sliver > https --lport 8443
```

This happens if Sliver's systemd service restarted without saving listener state. Just re-run the `https` command and you are back.

---

## Summary

Here is what you did today and why each step mattered:

| Step | What It Does | Why It Matters |
|------|-------------|----------------|
| Resource Group | Container for all resources | Clean teardown when engagement ends |
| VNet + Subnet (before VMs) | Network foundation | Avoids 2026 portal UI confusion |
| NAT Gateway | Fixes private subnet, gives VMs outbound internet | Without this, apt/Sliver install fails |
| NSGs | Cloud-level firewall | First layer of protection, before packets reach the VM |
| Password auth + fail2ban | VM access with brute force protection | NSG limits who can try; fail2ban limits retry attempts |
| Serial Console | Backup access if SSH breaks | Prevents total lockout |
| Sliver install + systemd | C2 server running persistently | Survives reboots |
| HTTPS listener on 8443 | Receives forwarded implant traffic | The core C2 channel |
| UFW on both VMs | OS-level firewall, second layer | Catches NSG misconfigurations |

---

**Next Step:** [02_domain_redirector.md](./02_domain_redirector.md)

Setting up your domain in Hostinger, getting a real SSL certificate with Certbot, and configuring Nginx to forward implant traffic to Sliver.
