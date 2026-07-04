# Day 2: Setting Up Your Sliver C2 Server on Azure (YouTube Script in Tenglish)

---

## 🎙️ Video Intro & Recap

welcome back to second video of our Red Team Infrastructure series.

last video lo manam C2 concepts, Redirector architecture, 
and OPSEC rules gurinchi matladukunnam right.

so ee video lo mission enti ante:
Azure Portal into jump ayyi, 
**mana backend C2 Server (`c2-server`) ni deploy & full ga harden cheddam.**

today video konchem technical untadhi, so carefully follow avvandi.

and one thing, servers create cheyadaniki
aws,digitial ocean, linode, gcp lanti providers unnapudu
Azure eh endhuku ante?

simple answer, na daggara azure lo 
free credits 100$ unnnai anthey!

so andhukani azure use chesthunna.

---


---

## 🛠️ Step 1: Find Your Operator Public IP

first step is very simple.

mana local terminal open chesi, mana operator machine public IP ento kanukkovali.

```bash
curl -s https://ifconfig.me
```

endhuku ante?

Azure firewall rules lo mana public ip ni  set chestham
so that only we can access it anamata.

note this ip somewhere down!
nenu na PUBLIC IP ni blur chesthunna
because of opsec.

---

## 🌐 Step 2: Create Resource Group & VNet

now Azure Portal open cheddam 
you can go to (`portal.azure.com`).

genreally meeru mee card tho signup chesthey100$ free credit vasthundhi
its more than enoughfor this.

repu real engegements lo aah infra motham
meeku company eh provide chesthundhi.

### 1. Create Resource Group
* So first thing, Go Search bar and type **Resource groups** 
open it and then click **+ Create**.
* mana group Name: `c2-infra-rg`
* Region: `india` (or mee choice). mee client ye contry lo untaro
aah location pedthey best because of good latency.
* Click **Review + create** -> **Create**.

sso ee  resource group endhuku ante, mana future lo
create cheyaboye vms, firewalls rules anni indhulo store ayyi untai

so its easy to manage all together at one place

### 2. Create Virtual Network
* Search bar lo **Virtual networks** -> **+ Create**.
* **Basics tab** lo:
  * Resource Group: `c2-infra-rg`
  * Name: `c2-vnet`
  * Region: `India` (or mee choice)
* **Address space tab** lo:
  * Azure by default IPv4 address range `10.0.0.0/16` and `default` subnet auto-populate chesthundhi.
  * Subnets box lo `default` row pakkana ఉన్న pencil icon ✏️ (Edit) click chesthe right side **Edit subnet** panel open avthundhi.
  * **Edit subnet** side panel lo:
    * Name: `default` (or `c2-subnet`)
    * Starting address: `10.0.0.0`, Size: `/24 (256 addresses)`
    * Private subnet section lo check **Enable private subnet (no default outbound access)**.
    * Click **Save** at the bottom of the panel.
* Click **Review + create** -> **Create**. 

azure lo from march 2026 nundi kothaga create chese subnets default ga private ga untai anamata.
ante outbound internet access undadhu.

so vms create chesaka meeru `apt update` chesina or tools install chesina
hang aypothadhi!
andhukani next step lo NAT gateway setup chestham for giving internet access.

---

## 🔌 Step 3: Create & Link Azure NAT Gateway

so ipudu mana private subnet ki internet access ivvadaniki NAT Gateway create chesthunam.

1. Search bar lo **NAT gateways** -> **+ Create**.
2. Resource Group: `c2-infra-rg`
3. Name: `c2-nat-gateway`
4. SKU: `Standard V2`
5. **Outbound IP** tab lo:
   click **Add public IP** -> create new IP named `c2-nat-pip`.
6. **Networking** tab lo:
   Virtual network: `c2-vnet` select cheyandi.
   Subnets: `default (10.0.0.0/24)` or `c2-subnet` select chesi check box tick cheyandi.
7. Click **Review + create** -> **Create**.

cool! ipudu mana private subnet lo vms ki outbound internet vasthundhi.

---

## 🔒 Step 4: Create C2 Network Security Group (NSG)

next, azure level firewall rule create cheddam.

1. Search bar lo **Network security groups** -> **+ Create**.
2. Name: `c2-server-nsg`
3. Resource Group: `c2-infra-rg` select chesi create cheyandi.
4. deployment complete ayyaka, go to **Inbound security rules** -> click **+ Add**:
   * Source: `IP Addresses`
   * Source IP: enter your public IP (apudey note cheskomanna IP)
   * Service: `SSH`
   * Destination Port: `22`
   * Protocol: `TCP`
   * Action: `Allow`
   * Priority: `100`
   * Name: `Allow-SSH-OperatorOnly`
5. Click **Add**.

note: manam inka port 8443 allow cheyaledhu because redirector IP inka create avaledhu kabatti.
adhi next redirector VM setup chesaka add chestham.

---

## 🖥️ Step 5: Provision C2 Server Virtual Machine

now main C2 Server VM deployment!

1. Search **Virtual machines** -> **+ Create** -> **Azure virtual machine**.
2. **Basics Tab**:
   * Resource group: `c2-infra-rg`
   * Name: `c2-server`
   * Image: `Ubuntu Server 22.04 LTS`
   * Size: `Standard_B2s` (2 vCPU, 4 GiB RAM)
   * Authentication: `Password` (username `c2admin` & strong password pedthunna)
   * **Public inbound ports**: Select **None**!
     *(ikada strict ga NONE select cheyandi, lekpothe azure SSH port ni overall internet ki open chesthundhi, which is bad for opsec!)*
3. **Networking Tab**:
   * Virtual network: `c2-vnet`
   * Subnet: `default`
   * Public IP: Create new `c2-server-pip` (Static)
   * NIC Network Security Group: Select **Advanced** -> select `c2-server-nsg` which we created earlier.
4. **Monitoring Tab**:
   * Enable **Boot diagnostics with managed storage account**
     *(idi endhukante in case firewall misconfiguration valla SSH lock ayithe browser nundi Serial Console access ravadaniki)*.
5. Click **Review + create** -> **Create**.

VM deployment complete ayyaka overview page lo unna `C2_SERVER_IP` note cheskondi!

---

## 🛡️ Step 6: System Hardening & Dependency Fixes

now mana local terminal nundi C2 VM ki SSH connect avvadam:
generally nenu terminal prefer cheyanu to connect  vms.

i used a software called termius. Idhi easy to manage anamata.
But its upto you.

```bash
so ssh c2admin@C2_SERVER_IP
```

### 1. Test Outbound Internet Access

first internet access undha ledha anedhi chechk cheddam 
```bash
ping -c 4 8.8.8.8
```
Perfect, were getting call back
so NAT gateway proper ga work avthundhi anamata!

### 2. Fix Ubuntu 22.04 `minisign` Dependency Issue
nenu first time sliver install chesinappudu `E: Unable to locate package minisign` ani error vachindhi.
why ante Ubuntu 22.04 default repos lo `minisign` package undadhu!

so script fail avvakunda direct ga universe repo & manual binary install chesthunam:

```bash
# Enable universe repo
sudo add-apt-repository universe -y && sudo apt update

# Install minisign binary manually
curl -sSL https://github.com/jedisct1/minisign/releases/download/0.11/minisign-0.11-linux.tar.gz | tar xz
sudo mv minisign-linux/x86_64/minisign /usr/local/bin/
sudo chmod +x /usr/local/bin/minisign
rm -rf minisign-linux
```

### 3. Install fail2ban
```bash
sudo apt install -y fail2ban
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo systemctl restart fail2ban
```

### 4. Configure UFW Firewall
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from YOUR_OPERATOR_IP to any port 22 proto tcp
sudo ufw allow 8443/tcp
sudo ufw enable
```
prompt vasthey `y` type cheyandi.

---

## ⚡ Step 7: Install Sliver C2 & Enable Service

now main thing, Sliver C2 install cheddam!

```bash
curl https://sliver.sh/install | sudo bash
```

installation complete ayyaka systemd service start chesi enable cheddam:

```bash
sudo systemctl daemon-reload
sudo systemctl enable sliver
sudo systemctl start sliver
sudo systemctl status sliver
```

`active (running)` vasthey setup clean ga install ayinattu!

---

## 🎧 Step 8: Start HTTPS Listener on Port 8443

now interactive sliver console open cheddam:

```bash
sudo sliver
```

sliver prompt `sliver >` raganey:

```
sliver > jobs
```

if empty ga unte, internal HTTPS listener start cheystam:

```
sliver > https --lport 8443
```

output:
```
[*] Starting HTTPS :8443 listener ...
[!] Successfully started job #1
```

---

## 🧪 Step 9: Local Verification

second SSH tab open chesi C2 server lo test cheddam:

```bash
curl -kv https://127.0.0.1:8443
```

`< HTTP/1.1 404 Not Found` ani vasthey **PERFECT**!
that 404 response means sliver HTTPS listener is live and rejecting raw non-implant requests.

---

## 🎬 Outro

so yeah, day 2 target complete ayindhi!

mana C2 server deploy ayyindhi, firewall hardening complete, and internal 8443 listener is running live.

next video lo Hostinger lo domain register chesi, Azure lo Redirector VM spin up chesi, Nginx + SSL setup cheddam!

so yeah that's it for this video, see you in next video!
