# Day 1: The Briefing

## The Situation

Monday morning. First day as a junior red team operator.

Your team lead calls you into a room. There is a whiteboard with three boxes connected by arrows.

```
TARGET COMPUTER --> [???] --> C2 SERVER
```

"Sit down," he says. "This is your first real engagement. By Friday, you will understand what that middle box is, why it exists, and how to build it."

---

## What We Are Doing

The client is a financial company. They have around 5,000 computers. They have a security team (called a SOC or blue team) that watches their network 24 hours a day.

Our job is to simulate a real attacker. We try to get inside their network, move around, and reach their important systems. We do this without the blue team catching us.

Your job this week is to build the infrastructure that keeps us hidden while we do all of this.

---

## What is C2?

C2 stands for Command and Control.

Here is the simple version:

When our malware (called an implant) runs on a computer inside the client's network, it needs to talk to us so we can send it instructions. C2 is that communication channel between us (outside) and the implant (inside).

**Why can't we just connect to the implant directly?**

Because the client's firewall blocks all incoming connections from the internet. Nothing from outside can reach inside.

But the firewall does allow outgoing connections. Employees need to browse websites, use email, download updates. So the firewall lets traffic go from inside to outside.

This means our implant has to be the one that starts the connection. It reaches out from inside the network to our server on the internet. This is called a reverse connection.

**How the loop works:**

1. Implant runs on target computer
2. Every 60 seconds, implant connects to our server
3. Implant asks: "Any commands for me?"
4. We send a command (example: "list all files in C:\Users")
5. Implant runs the command and sends us the output
6. Implant disconnects and waits 60 seconds
7. Loop repeats

Everything we do on the engagement goes through this loop.

**What commands can we send?**

- Run any command on the target (like opening a terminal and typing commands)
- Take a screenshot of what the user sees
- Download files from the target to our server
- Upload tools from our server to the target
- Extract saved passwords from the computer
- Jump from one compromised computer to other computers on the same network

If the C2 channel goes down, we lose control of everything. That is why building it properly matters so much.

---

## The Wrong Way to Set Up C2

Here is how most beginners do it. And why it fails.

```
Target Computer --> internet --> Your C2 Server (IP: 13.89.45.22)
```

You rent a VM on Azure. You install Sliver (the C2 tool we use). You tell the implant to connect to your server's IP address. The implant runs on the target. It connects back. You have a C2 channel.

**What happens next on a real engagement:**

The blue team's monitoring system (called a SIEM) collects logs of every connection the target computers make to the internet.

An analyst looks at the logs and sees this:

> Computer FIN-WKS-047 is connecting to IP 13.89.45.22 every 60 seconds. Each connection lasts 2 seconds. The amount of data sent is the same every time.

This is not normal. When a real person browses the internet, connections happen in bursts. They click a link, 10 connections happen at once, then nothing for 3 minutes while they read. Then another burst.

A computer connecting to the same IP every 60 seconds with the same amount of data is clearly a program, not a person. The blue team calls this "beaconing" and it is one of the strongest signs of malware.

The analyst then visits your IP in her browser. Sliver's default setup shows a suspicious TLS certificate. She blocks your IP at the firewall.

Your implant can no longer connect. The C2 channel is gone. Engagement is over.

**Time from first connection to getting caught: 2 to 4 hours.**

And here is the worst part. The implant has your real server's IP hardcoded inside it. Now the blue team has your real IP. That server is burned forever. You cannot use it for this engagement or any future engagement.

---

## The Right Way: The Redirector

We put a separate server between the target and our real C2 server.

```
Target --> Redirector (Public IP) --> C2 Server (No public IP)
```

The middle server is called a redirector. Its only job is to receive connections from implants and pass them to the real C2 server.

**Why does this help?**

The implant only knows the redirector's address. Not the real C2 server. If the blue team finds and blocks the redirector, we just create a new one. The real C2 server stays safe and running the whole time.

**How the redirector filters traffic:**

The redirector runs Nginx (a web server software). Nginx looks at every connection it receives and checks:

- What URL path is the request going to? (Example: `/api/v1/status`)
- What is the User-Agent? (This is a header that says who is making the request. Chrome sends one value. Our implant sends a custom value we choose.)
- Are any custom headers present? (We can tell our implant to send a secret header that only we know about.)

If all three match our implant's pattern, Nginx passes the traffic to the real C2 server.

If anything does not match (like a blue team analyst browsing the address in Chrome), Nginx sends them to Microsoft's website instead. The analyst sees Microsoft's homepage and thinks the IP belongs to some Microsoft service.

```nginx
# If the request matches our implant pattern, forward it to C2 server
location /api/v1/status {
    if ($http_user_agent != "Mozilla/5.0 (Windows NT 10.0) OurCustomString") {
        return 403;
    }
    proxy_pass https://10.0.0.1:8443;
}

# Everything else goes to Microsoft
location / {
    return 302 https://www.microsoft.com;
}
```

What each part does:

- `location /api/v1/status` means: only handle requests going to this URL path
- `if ($http_user_agent != ...)` means: if the User-Agent does not match our implant's value, block the request
- `proxy_pass https://10.0.0.1:8443` means: send matching requests to our real C2 server
- `return 302 https://www.microsoft.com` means: send everyone else to Microsoft

**What the blue team sees now:**

The same analyst investigates. She visits the redirector IP in Chrome. Nginx sees a Chrome User-Agent going to `/`. It does not match. She gets redirected to Microsoft. She thinks it is a Microsoft service and moves on.

Even if she blocks the redirector IP anyway, we create a new redirector VM in 5 minutes and point our domain to the new IP. The implant reconnects automatically. Total downtime: a few minutes.

And the real C2 server never appeared in any of her logs. It has no public IP. It is invisible.

---

## The Full Setup We Are Building

```
Implant --> Redirector (Nginx + SSL) --> WireGuard Tunnel --> C2 Server (Sliver)
```

**Part 1: The Implant**

Runs on the target computer. Connects to our domain name. Uses HTTPS on port 443 (same port as normal websites). Checks in every 60 seconds with random variation so it does not look like a timer. Has no information about the real C2 server.

**Part 2: The Redirector**

A VM on Azure with a public IP. Runs Nginx. Has a real SSL certificate (we get this free from Let's Encrypt using a tool called Certbot). Filters traffic. Passes implant traffic through. Shows everyone else a fake page. This is the only server that can be seen from the internet.

**Part 3: The WireGuard Tunnel**

An encrypted connection between the redirector and the C2 server. WireGuard is VPN software. It creates a private encrypted tunnel between two computers. The redirector sends implant traffic through this tunnel to the C2 server. Nobody watching the network between these two servers can read what is inside the tunnel.

**Part 4: The C2 Server**

Runs Sliver. Has no public IP. Cannot be reached from the internet at all. Only accepts traffic coming through the WireGuard tunnel from the redirector. This is where we sit and control everything.

---

## Why We Use Sliver

Sliver is a free and open-source C2 tool made by a security company called BishopFox. It is one of the best free options available today.

The most famous C2 tool is Cobalt Strike. But it costs thousands of dollars per year. And because it has been used so long, security companies have built many ways to detect it.

Sliver is newer. It supports multiple connection types: HTTPS, DNS, and WireGuard. It can run multiple operators at the same time (so your whole team can connect). It supports BOFs (small programs that run inside the implant without creating new files on disk). Russian hacking group APT29 has been confirmed using Sliver in real attacks, which tells you it is production-grade.

The default Sliver settings are known to security vendors now too. So we customize everything: the URL paths, the User-Agent, the HTTP headers, and the TLS settings. We cover this in the later files.

---

## How the Blue Team Detects C2 (And How We Beat Each One)

**Detection 1: Beaconing (Regular Timing)**

The SIEM notices a computer connecting to the same IP every 60 seconds exactly. This regular pattern is a clear sign of a machine, not a human.

Our defense: We add jitter. In Sliver, we set the base sleep time to 60 seconds and jitter to 30 percent. The implant sleeps a random amount between 42 and 78 seconds each time. 60 seconds average, but irregular enough to not trigger timing alerts.

**Detection 2: TLS Fingerprinting**

When a computer connects over HTTPS, the first message it sends (called Client Hello) contains details about what encryption it supports. Different programs send different values. Security tools compute a fingerprint (called JA3) from these values. The default Go programming language TLS settings (which Sliver uses) have a known JA3 fingerprint that security tools flag.

Our defense: We customize the TLS settings in Sliver's C2 profile to produce a different fingerprint that does not match the known Sliver signature.

**Detection 3: URL Patterns**

Sliver's default URL paths (like `/api/v1/sync.woff`) are published in security research as Sliver indicators. If the blue team can see your HTTP traffic, they will recognize them.

Our defense: We change all the default URL paths in a custom C2 profile. We use paths that look like normal API traffic.

**Detection 4: Suspicious Domain**

Domains registered yesterday with no website content look suspicious. The SIEM flags connections to brand new domains.

Our defense: We register the domain at least 30 days before the engagement. We put some fake content on it so it looks like a real service.

**Detection 5: Endpoint Monitoring (EDR)**

The client runs security software on every computer (called EDR, like Microsoft Defender for Endpoint). It watches every process, every network connection, every file created. If our implant does something suspicious like writing to disk in unusual places or allocating memory with execute permissions, EDR raises an alert.

Our defense: We use Sliver's evasion features. The `--evasion` flag removes security hooks that EDR places on Windows. We can also inject our implant into a trusted process (like a browser) so the EDR sees normal software making network connections, not a suspicious new program.

---

## OPSEC Rules

OPSEC means Operational Security. These are the rules we follow to not get caught.

**Rule 1: The C2 server never gets a public IP.**
It is invisible to the internet. The only way to reach it is through the WireGuard tunnel from the redirector.

**Rule 2: Change every default in Sliver.**
The defaults are published and known. URLs, headers, User-Agent, TLS settings. Change all of them.

**Rule 3: The redirector is disposable.**
If the blue team finds it, we delete it and make a new one in 5 minutes. Never store anything important on it.

**Rule 4: Test everything before the engagement.**
Run the implant on a test Windows VM. Use netstat to check it only connects to the redirector. Use Wireshark to check the traffic looks like normal HTTPS. Visit the redirector in a browser to confirm decoy works.

**Rule 5: Fresh infrastructure for every engagement.**
Never reuse domains, IPs, or certificates between different clients. Delete everything when the engagement ends.

**Rule 6: If you are not sure, ask.**
Fixing a mistake now costs minutes. Fixing it after the engagement starts can cost the whole operation.

---

## Real Examples of C2 Getting Caught

**Case 1:**
A red team sent a phishing email. The employee who got it uploaded the attachment to VirusTotal (a website that scans files). VirusTotal shared it with security companies. Within hours, every antivirus flagged the implant. Security researchers pulled the C2 domain from the implant file and started scanning the infrastructure. The domain was burned before the engagement even started.

**Case 2:**
Security researchers scanned the entire internet looking for servers using Sliver's default settings: default ports, default certificates, default URL paths. They found hundreds. Every one was identified, catalogued, and added to blocklists.

**Case 3:**
A developer at the North Korean hacking group Lazarus had his laptop infected with info-stealing malware. The malware stole his saved passwords and files, including credentials for their C2 infrastructure. Investigators used this to map out Lazarus's entire operation.

---

## Your 3-Day Plan

### Day 1: Set Up the C2 Server

- Spin up an Ubuntu 22.04 VM on Azure. No public IP.
- Secure SSH (key-only, no passwords).
- Install Sliver.
- Install WireGuard. Configure the tunnel (10.0.0.1 on C2 side).
- Start an HTTPS listener in Sliver bound to the WireGuard tunnel IP.
- End state: Sliver running, waiting for connections, invisible to the internet.

### Day 2: Set Up the Redirector and Domain

- Register a domain in Hostinger. Pick a name that sounds like a real service.
- Spin up a second Ubuntu 22.04 VM on Azure. This one gets a public IP.
- Install Nginx and Certbot.
- Get a real SSL certificate from Let's Encrypt using Certbot.
- Write the Nginx config to filter traffic and forward to C2 server.
- Configure WireGuard on the redirector side (10.0.0.2). Connect the tunnel.
- Point the domain's DNS A record to the redirector's public IP.
- End state: domain points to redirector, redirector forwards to C2 server through tunnel.

### Day 3: Generate Payload and Test

- Create a custom C2 profile in Sliver (change default URLs and headers).
- Generate a beacon payload for Windows.
- Run the payload on a test Windows VM.
- Check Sliver console for the beacon connecting in.
- Run netstat on the test VM: confirm it only connects to the redirector IP, not the C2 server.
- Open a browser and visit the redirector domain: confirm you see the decoy (Microsoft redirect).
- End state: full chain working. Implant to redirector to tunnel to C2 server. Ready for engagement.

---

## Tools We Use

| Tool | What It Does | Where It Runs |
|---|---|---|
| Sliver | C2 framework, generates implants, controls compromised computers | C2 server VM |
| Nginx | Web server that filters and forwards traffic | Redirector VM |
| WireGuard | Creates encrypted tunnel between redirector and C2 server | Both VMs |
| Certbot | Gets free SSL certificates from Let's Encrypt | Redirector VM |
| Azure | Cloud provider where we rent our VMs | Web browser |
| Hostinger cPanel | Where we manage our domain and DNS records | Web browser |
| Wireshark | Captures network traffic so we can inspect it | Our local machine |
| netstat | Shows active network connections on a computer | Any machine |

---

## Terms (Simple Definitions)

**Implant:** Our program running on the target computer. Also called a beacon or agent.

**Beacon:** An implant that checks in on a schedule (every 60 seconds) instead of staying connected the whole time.

**C2 Server:** Our server that the implant connects to. We control it.

**Redirector:** A server that sits in front of the C2 server. The implant connects to the redirector. The redirector passes the traffic to the C2 server. Hides our real server.

**Listener:** A service on the C2 server that waits for implant connections.

**SOC:** Security Operations Center. The client's security team that monitors for attacks.

**EDR:** Software on a computer that watches for suspicious activity (like Microsoft Defender for Endpoint).

**SIEM:** Software that collects logs from all devices on a network and generates alerts.

**WireGuard:** VPN software that creates an encrypted tunnel between two computers.

**Nginx:** Web server software. We use it as a reverse proxy on the redirector.

**Certbot:** Tool that gets free SSL certificates from Let's Encrypt automatically.

**HTTPS:** Encrypted web traffic. Uses port 443. Looks like normal website traffic.

**Jitter:** Random variation added to the implant's check-in timer so it does not create a regular pattern.

**JA3/JA4:** Fingerprints based on how a program sets up HTTPS connections. Used to identify specific software.

**Reverse Connection:** When the implant starts the connection to us, instead of us connecting to it. This works because firewalls block incoming connections but allow outgoing ones.

**Firewall:** A system that controls what network traffic is allowed in and out of a network.

**Beaconing:** When a program makes regular timed connections to an IP. A strong sign of malware to the blue team.

**Burn:** When the blue team finds and blocks part of your infrastructure. "The redirector got burned."

---

## Next Step

Continue to: [01_setup_sliver_c2.md](01_setup_sliver_c2.md)
0