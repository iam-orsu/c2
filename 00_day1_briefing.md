# Day 1 Briefing: Your First Red Team Engagement

---

## Before We Start

Listen, sit down. Close that browser tab you have open on "top 10 red team tools." We are not doing that today.

Today is your first real engagement. Not a CTF. Not a lab. A real company with a real security team watching real traffic on a real network. They have EDR on every endpoint. They have a SIEM collecting logs from every server. They have a SOC with analysts sitting and looking at dashboards all day. Their job is to catch people like us.

Your job, starting today, is to not get caught.

I am going to walk you through exactly what we are building over the next three days, why every single decision we make matters, and what happens when you get lazy and skip steps. I have seen junior operators burn entire engagements because they thought a step was optional. Nothing in what I am about to tell you is optional.

We are setting up a Command and Control infrastructure from scratch. By the end of Day 3, you will have a working C2 setup that can receive connections from a compromised Windows machine without the blue team knowing where our real server is. That is the goal. Let me explain every piece of it.

---

## What is Command and Control, Really?

Let's start from zero, because I want you to understand this at a deep level, not just repeat a definition.

When you compromise a machine during a red team engagement, what does that actually mean? You found a vulnerability. You exploited it. Something ran on their machine. Now what?

The answer is: nothing, unless you have a way to talk to that machine and tell it what to do next.

You cannot just walk up to the target machine. You cannot SSH into it from outside, because their firewall blocks inbound connections. You cannot leave a backdoor on a random port and expect it to stay open, because their firewall will block that too and their EDR will flag the listening process.

So here is the actual problem you need to solve: **you need a persistent, stealthy, two-way communication channel between a machine inside their network and a machine you control outside their network.** That communication channel is C2. Command and Control.

This is what C2 does at a technical level:

1. You compromise a machine. Something runs on it. That "something" is the implant (also called an agent, a beacon, a stager, or a payload depending on context).
2. The implant is a program. It runs as a process. It is compiled code sitting in memory on the victim machine.
3. The implant makes outbound network connections. It calls home to a server you control.
4. On that server, you have C2 framework software (in our case, Sliver) listening for those connections.
5. Sliver receives the check-in from the implant. It now has a "session" or a "beacon" registered.
6. You, sitting at your operator machine, open the Sliver console and see that beacon. You type commands.
7. Sliver queues those commands.
8. On the next check-in, the implant grabs the command queue, runs the commands on the victim machine, collects the output, and sends it back.
9. You see the output in your console.

That is the full loop. From your keyboard to their machine and back.

Now, why does this work? Why does the firewall not block it?

### Why Outbound HTTPS is Different

Most corporate firewalls are configured to block unsolicited inbound traffic. If an outside machine tries to connect TO something inside the network on a random port, the firewall stops it. That is basic firewall logic.

But outbound traffic is a different story. When an employee's machine opens a web browser and visits a website, that is an outbound connection. The machine inside the network reaches out to a server on the internet. The firewall allows this. If it did not allow it, no employee could browse the internet.

Your implant does the same thing. It makes an outbound HTTPS connection to a server on the internet. From the firewall's point of view, this looks identical to a browser visiting a website. The connection goes out on port 443, it is encrypted (HTTPS), and it goes to some domain on the internet.

The firewall sees: `internal machine ---> HTTPS ---> some domain`. The firewall allows it.

This is why C2 traffic runs over HTTPS. Not because we are worried about encryption (though that matters too), but because HTTPS on port 443 is the most allowed, most common, most expected type of outbound traffic on any corporate network. We hide in plain sight.

### Beaconing vs Sessions

Now here is something important that a lot of beginners get confused about. There are two types of implant behavior in Sliver: **Beacons** and **Sessions**.

**Sessions** keep a persistent, real-time connection open. The implant connects to the C2 server and stays connected. You send a command, it responds immediately. This is fast and interactive, but it is also loud. A permanent connection from an internal machine to an external server that stays open for hours or days is going to get flagged by network monitoring tools. Real-time connection analysis and anomaly detection will catch this.

**Beacons** are completely different. A beacon does not maintain a persistent connection. It sleeps, wakes up, makes one quick HTTPS request to the C2 server to check if there are any commands waiting, runs any commands it finds, sends the results back in another HTTPS request, and then goes back to sleep. The connection is open for seconds, then gone.

For example, if you set a beacon interval of 60 seconds with 20% jitter:
- The beacon sleeps for somewhere between 48 and 72 seconds (random within that range)
- Wakes up, makes an HTTPS POST request to the C2 server
- Grabs any queued commands
- Executes them
- Sends results back
- Sleeps again

From the network's perspective, it looks like a machine making occasional HTTPS requests to a web server. Nothing unusual. No persistent connection. No port that is "always open."

The jitter (the randomness in timing) is also important. If the beacon checked in every exactly 60 seconds, a tool like Rita or Zeek would detect that as a regular beaconing pattern. Network traffic from real users is not on a timer. Adding jitter makes the interval look random, which is what real traffic looks like.

**For this engagement, we use beacons. Not sessions.** Sessions are for when you need interactivity right now and have already established a solid foothold. On initial access and long-term persistence, always use beacons.

---

## The Sliver C2 Framework

We are using Sliver for this engagement. Let me explain what it is, why we chose it, and what makes it different from other frameworks. I do not want you just running commands without knowing why.

### What Sliver Is

Sliver is an open-source C2 framework built by Bishop Fox, a professional offensive security firm. Bishop Fox does real red team work for some of the largest companies in the world. They built Sliver to replace Cobalt Strike on their own engagements. The code is on GitHub. Anyone can read it, anyone can use it, and Bishop Fox's team actively maintains it.

It supports multiple communication protocols:
- **HTTPS**: What we will use. Traffic over port 443, looks like web traffic.
- **HTTP**: Unencrypted, never use this in a real engagement unless you specifically need it for a reason.
- **DNS**: C2 traffic tunneled inside DNS queries. Extremely slow but almost impossible to block without breaking DNS entirely. Used when HTTPS is blocked.
- **mTLS (Mutual TLS)**: Both sides verify certificates. Very secure, but unusual enough that it can stand out.
- **WireGuard**: VPN-like tunneling. Great for operator connectivity to the team server.

For a first engagement with standard access, HTTPS is what you want. It blends in. It is fast enough. It goes through proxies. It works.

### Why Not Cobalt Strike?

You will hear about Cobalt Strike constantly in this industry. It has been the dominant commercial C2 framework for years. But there are real problems with using it now:

**Detection signatures are everywhere.** Cobalt Strike has been around since 2012. Every major EDR vendor (CrowdStrike, SentinelOne, Microsoft Defender, Carbon Black) has hundreds of Cobalt Strike signatures. The default Cobalt Strike configuration is detected instantly on modern endpoints. To use it effectively now, you need extensive customization, and even then you are fighting against 13+ years of defensive signatures.

**The cracked versions destroyed its stealth value.** A cracked version of Cobalt Strike was leaked and has been used by actual criminal groups and nation-state actors for years. This means defenders have seen real-world Cobalt Strike traffic extensively. The detection rate is extremely high.

**Cost.** Cobalt Strike costs over $3,500 per year per license. For a firm running multiple concurrent engagements, that adds up.

Sliver is newer. Detection coverage is growing but is not nearly as comprehensive as Cobalt Strike detection. More importantly, Sliver has an architectural advantage that matters a lot:

### Per-Implant Encryption Keys

This is something that genuinely makes Sliver better for engagements. Every time you generate a Sliver implant (payload), Sliver compiles a fresh binary with a unique asymmetric key pair embedded in it. Public key goes in the implant, private key stays on the team server.

Why does this matter? In Cobalt Strike, if a blue team captures your beacon, they can extract the watermark and encryption keys from it. Those keys are potentially shared across your campaign. They can decrypt all your C2 traffic, retroactively. They can identify every beacon associated with your team server.

With Sliver, if they capture one implant and reverse engineer it, they get the keys for that one implant. The other implants on other machines have completely different keys. Capturing one does not help them with the others. Each implant is cryptographically isolated.

This is not a small thing. In long engagements where you have 10-15 compromised machines, this is the difference between losing one foothold and losing everything.

### The Operator/Server Model

Sliver runs as a server daemon on your C2 VM. You, the operator, connect to it using a Sliver client. When you type commands in the Sliver console, the client sends those commands to the server, which queues them for the relevant beacon.

Multiple operators can connect to the same Sliver server at the same time. You and your team lead can both be in the console, both looking at beacons, collaborating. This is how professional red team firms work. You have a shared server with shared visibility.

The operator client connects to the Sliver server over mTLS with an operator certificate. The server generates this certificate for you. Without the certificate, nobody can connect to the operator port. This means even if someone finds your C2 server, they cannot interact with Sliver unless they have the operator cert.

---

## Why We Need a Redirector

This is the most important architectural decision we are making, and I want you to understand it deeply before we touch a single command.

### The Problem With a Direct Setup

If you set up Sliver on an Azure VM at some IP address, and you generate an implant that connects directly to that IP, here is what the traffic flow looks like:

```
Victim machine  ---HTTPS--->  Azure VM (Sliver C2 server)
                                 IP: 20.100.50.200
```

Now the victim machine is running your implant. It is making regular HTTPS requests to `20.100.50.200`. The blue team is monitoring network traffic (they are). Here is what they see in their SIEM logs:

```
Source: 192.168.10.55 (internal workstation)
Destination: 20.100.50.200 (external IP)
Port: 443
Protocol: HTTPS
Frequency: every ~60 seconds
```

That repeated connection to the same external IP, at a regular interval, from an internal workstation, is going to get flagged. Their SIEM has rules for this. Their network monitoring tool (maybe Zeek with Rita, maybe Darktrace, maybe whatever they have) watches for exactly this pattern. Regular intervals, consistent destination, ongoing over hours.

Now the analyst who gets the alert investigates. They look up `20.100.50.200`. Azure IP, no legitimate business relationship with this company, domain is blank or has no content, SSL certificate is weird. They do a JARM scan on it (a tool that fingerprints TLS servers by how they respond to specific TLS handshake sequences). The JARM hash matches Sliver's default TLS configuration. Instant confirmation.

They block `20.100.50.200` at the firewall. They kill the session. They start tracing which machines connected to that IP and when, to find out how far you got.

But it gets worse. Your C2 server is now compromised from a detective standpoint. They have your server's IP. They can:

- Report it to Azure abuse. Azure shuts it down.
- Monitor all future connections to it (none will come now that it's blocked).
- Pull threat intelligence on the IP. They find your JARM hash, your TLS certificate details, any prior scanning or reconnaissance you did from that IP.
- If you were sloppy about infrastructure, they might correlate this IP with other infrastructure you used in the engagement.

Your entire operation is over. You have to rebuild from scratch, which costs you days.

### What a Redirector Solves

A redirector is a separate server that sits in front of your C2 server. It receives the implant's connections and forwards them to the real C2 server. The implant never has your real C2 server's IP in it. It only has the redirector's IP (or domain).

Traffic flow with a redirector:

```
Victim machine  ---HTTPS--->  Redirector VM  ---HTTP--->  C2 Server (Sliver)
                              (Nginx proxy)              (hidden, not internet-facing)
                              your domain
                              real SSL cert
```

Now what does the blue team see? They see HTTPS connections from the workstation to your domain. They investigate that domain. They hit the redirector. The redirector runs Nginx and serves a normal-looking page to anyone who just browses to it. The SSL certificate is a real Let's Encrypt certificate, not self-signed. The JARM hash is Nginx's JARM hash, which matches millions of other web servers.

They run their tools. Nothing looks obviously malicious. They might still be suspicious, so they block the domain at the firewall. Here is what happens to you:

1. Your implant can no longer reach the redirector. That implant goes dark.
2. But your C2 server is completely fine. It never got touched.
3. You spin up a new redirector VM in 10 minutes.
4. You update the DNS record for your domain to point to the new VM.
5. Within a couple of minutes, DNS propagation happens and the implant (if it was not killed) starts connecting to the new redirector.
6. All your other implants on other machines that were not using that domain, completely unaffected.

Redirectors are designed to be burned. They are cheap (a couple of dollars a day on Azure). They are fast to set up. They are disposable. **Your C2 server is not disposable.** It holds all your sessions, all your collected data, all your tools, all your operator configurations. You protect it by never exposing it directly.

### What Nginx Actually Does as a Redirector

You are going to hear "Nginx reverse proxy" a lot. Let me tell you exactly what this means.

Nginx is a web server. When you configure it as a reverse proxy, it does this:

1. It listens on port 443 (HTTPS).
2. When a connection comes in, it looks at the request.
3. Based on rules you write, it either:
   - Forwards the request to the backend (your C2 server)
   - Serves a local response (the fake landing page for suspicious scanners)
   - Redirects to somewhere else

When the implant makes an HTTPS request to your domain:

```
Implant sends:
POST /update HTTP/1.1
Host: updates.yourdomain.com
User-Agent: [whatever profile you use]
Content-Type: application/octet-stream
[encrypted C2 traffic as request body]
```

Nginx receives this. It strips the outer HTTPS layer (because Nginx handles SSL termination). It looks at the request. It matches your proxy rule. It forwards the request (now plain HTTP) to your C2 server on a private port. Your C2 server processes it, generates a response, sends it back to Nginx. Nginx wraps it back up and sends it to the implant.

The implant thinks it is talking to Nginx's server. The blue team thinks it is talking to a normal web server. Only you know there is a Sliver server sitting behind it.

Your actual Sliver server never touches the internet directly. Its firewall (Azure Network Security Groups + UFW) only allows connections from:
1. The redirector's IP on the C2 port
2. Your operator IP on the SSH port (22)

Everything else is blocked. If someone tries to connect to your C2 server from the internet, they get nothing. The port is not even responding. The server looks like it does not exist.

### Smart Redirector Behavior

A basic redirector just forwards everything. A smart redirector filters traffic. We will set ours up to differentiate between C2 traffic and scanner traffic.

How? By checking request characteristics.

Legitimate Sliver beacons will make requests with specific URL patterns (the paths Sliver uses by default, or custom ones you configure). Random internet scanners, threat intelligence companies crawling your IP, and curious blue teamers manually probing your domain will make different kinds of requests.

We can configure Nginx to:
- Forward requests that match Sliver's URL patterns to the C2 server
- Serve a normal-looking landing page to everything else

This way, if Shodan or a threat intel crawler hits your domain, they see a boring webpage. They log it, move on, do not flag it. If your implant hits your domain, it gets forwarded to Sliver.

---

## How Blue Teams Actually Detect C2

I want you to understand exactly what you are up against. These are the real techniques blue teams use. Every architectural decision we make in the next two days is a response to one of these.

### Technique 1: Beaconing Detection with Frequency Analysis

Tools like Rita (Real Intelligence Threat Analytics) and Darktrace watch network traffic and look for connections that happen at a suspiciously consistent interval. Here is what that analysis actually does:

It collects all network flow records (source IP, destination IP, port, timestamp) for a given time window. It groups flows by (source, destination, port). For each group, it calculates the statistical distribution of the time gaps between connections. If the distribution is tight (low standard deviation) meaning the gaps are almost always the same length, that is a beaconing signal.

A human browsing the internet has wildly variable intervals between requests. A beacon checking in every 60 seconds has extremely low variance. Rita gives it a high beaconing score. An analyst gets an alert.

**Our counter:** Set beacon interval to something reasonable (60-120 seconds) with jitter of at least 20-30%. The jitter adds randomness to the interval, which increases the statistical variance and makes it look less mechanical. We will set this when we generate our beacon.

### Technique 2: JARM Fingerprinting

JARM is a TLS server fingerprinting tool built by Salesforce security researchers. It works by sending 10 specially crafted TLS ClientHello packets to the target server and recording exactly how the server responds. The server's TLS stack (the library, version, cipher order, extensions) determines how it responds, and that creates a fingerprint hash called the JARM hash.

Sliver's built-in HTTPS listener (without a fronting proxy) has a known JARM hash. Defenders have catalogued the JARM hashes of major C2 frameworks. Tools like Shodan actively JARM scan the entire internet and index results. If your Sliver server's port 443 is open to the internet and has Sliver's JARM hash, it gets indexed and flagged.

**Our counter:** Two things. First, Sliver's HTTPS listener is NOT exposed to the internet. Its firewall blocks all internet access to it. Second, Nginx is what the blue team and scanners actually reach. Nginx has a completely standard JARM hash. It looks like any normal web server.

### Technique 3: TLS Certificate Analysis

Blue teams pull and analyze SSL certificates from any external server that internal machines connect to. They look for:

- Self-signed certificates (no trusted CA signed them, huge red flag)
- Certificates with suspicious subject details (weird organization name, no subject alternative names)
- Certificates issued very recently (issued today? For a server your company suddenly started connecting to?)
- Certificate reuse (same cert on multiple IPs, suggests shared infrastructure)

If your redirector has a self-signed certificate, the moment a blue team analyst pulls it, they know something is wrong. Real web servers serving legitimate traffic have certificates signed by a trusted CA.

**Our counter:** We use Certbot (Let's Encrypt) to get a real, CA-signed certificate for our redirector domain. Let's Encrypt is a free, trusted Certificate Authority. Their certificates look identical to the certificates used by millions of real websites. Nothing suspicious about it.

### Technique 4: Domain Reputation and Age

Every time a machine in the corporate network visits a new domain, the proxy or DNS logs record it. Security teams subscribe to threat intelligence feeds that categorize domains (business category, age, reputation, etc.).

A domain registered yesterday, never seen in internet traffic before, that your internal machine suddenly starts connecting to? High alert. Threat intel feeds classify it as uncategorized or newly registered. Some companies block all traffic to domains less than 30 days old.

**Our counter:** Domains are registered in advance by the engagement team. Not the day before. Weeks before. The domain should have some DNS history. Ideally, some basic web content was posted on it at some point so crawlers have indexed it.

### Technique 5: Network Scanning of Identified Infrastructure

Once a blue team identifies a suspicious IP or domain, they do not just block it. They actively investigate. They scan every port on that IP. They pull banners from every open service. They check it against Shodan's historical data.

If your C2 server has Sliver running on port 443 and it is internet-facing, a simple port scan reveals it. A banner grab shows Sliver's response. Game over.

**Our counter:** The C2 server has no internet-facing open ports beyond what your firewall explicitly allows. We use Azure Network Security Groups to drop everything by default, then allow only specific source IPs on specific ports. The C2 server is essentially invisible to the internet.

### Technique 6: EDR Memory and Process Analysis

This is endpoint-side detection, not network. EDR products (CrowdStrike Falcon, SentinelOne, Microsoft Defender for Endpoint) scan process memory in real time. They look for:

- Known shellcode patterns
- Executable code in memory regions marked as non-executable
- Process injection (code from one process appearing in another process's memory)
- Suspicious API call sequences (VirtualAlloc -> WriteProcessMemory -> CreateRemoteThread is the classic injection pattern)
- Unsigned code running in signed process memory space

Sliver implants can be detected by modern EDR if deployed naively. A raw EXE beacon dropped on disk and executed directly will likely get caught. Payload evasion is a separate, deep topic that goes beyond this course, but you need to know it exists.

**What we do for this engagement:** We are testing in an isolated lab environment. The Windows test VM does not have EDR. In a real engagement with a hardened target, you would use shellcode loaders, process injection, AMSI bypasses, and various obfuscation techniques. We will cover that in a future course. For now, understand the concept.

---

## Real Engagements Where Teams Got Caught

These scenarios are based on documented red team failures. Study them. Each one teaches something concrete.

### The Collapsed Infrastructure Problem

A red team ran a multi-week engagement. They used a single cloud provider account for their phishing server, their C2 server, and their file staging server (where they uploaded tools to victim machines). All three VMs were on the same Azure subscription and their IPs were in the same /24 subnet.

The blue team found the phishing page (it was detected because one employee reported the suspicious email). They looked at the phishing server's IP. They ran a scan of the entire /24 subnet that IP was in. Found two more VMs. Scanned both. Found the C2 server and the staging server. Blocked all three. Pulled logs from their DNS and firewall to find every machine that connected to any of these IPs.

The red team lost their entire operation in 40 minutes because of one phishing page being found.

**What this teaches you:** Infrastructure separation is not paranoia. Different components of your operation should be on different accounts, different providers, different IP ranges. If one gets burned, the others survive. For this engagement: your C2 server and your redirector are separate VMs. If you had a phishing component, it would be on a third, completely separate VM with no logical connection to the other two.

### The Default Tool Signature Problem

A team deployed Metasploit's Meterpreter on a Windows machine. They used the default payload with no customization. No custom User-Agent, no custom URI patterns, no encryption profile changes. Just default Meterpreter talking to their server.

The client's EDR had a signature for the exact memory pattern that Meterpreter creates when it injects into a process. The moment Meterpreter injected into `explorer.exe`, the EDR killed the process and raised an alert. The SOC had the workstation isolated within 6 minutes. They never even got to run a single post-exploitation command.

**What this teaches you:** Default tool configurations are known to defenders. The security community has been publishing Meterpreter detection techniques for 15+ years. The same is increasingly true for Cobalt Strike, and beginning to be true for Sliver. Never run tools with defaults in a real engagement. Always customize, and always test against a similar security stack if you can.

### The JARM Scan That Ended an Engagement

A red team consultant set up a Cobalt Strike team server on a DigitalOcean VPS. Port 443 was open to the internet (they thought they needed it for the beacon to connect). They did not use a redirector.

A threat intelligence researcher doing routine internet scanning ran a JARM scan against that IP range. The JARM hash matched Cobalt Strike's default profile. The researcher published it in a threat intel feed. The client's security team subscribed to that feed. The security team saw their own internal machines connecting to a known Cobalt Strike server. They called their vendor, who confirmed it was almost certainly a red team tool (not a real attack given the context), but the engagement was effectively compromised. The client required a full debrief and the methodology was flagged as insecure.

**What this teaches you:** Your C2 server port 443 should NEVER be open to the internet. The only machines that should reach it are your redirector's IP (for forwarding traffic) and your operator IP (for SSH). Everything else should be blocked at the firewall level. If you run a JARM scan against your C2 server from an external IP, you should get nothing back, because the port is not even responding to that IP.

### The Metadata Leak

A red team created a malicious macro-enabled Word document for a phishing campaign. They created it on a Windows machine where the user account was named `RT_Consultant_2024`. They did not check the document properties before sending. The document's Author metadata field was `RT_Consultant_2024`.

The blue team received the flagged email (one employee reported it). They opened the document in a sandboxed analyzer. The analyzer pulled all metadata from the document. The author field was right there. They Googled `RT_Consultant_2024`. Found a LinkedIn profile with a work history that included the red team consulting firm. Called the client's CISO. The CISO confirmed there was an active engagement. The element of surprise was gone.

**What this teaches you:** Every artifact you create, every file you compile, every document you write for an engagement, strip the metadata. Check it. Compile your tools on clean VMs where the username is generic. Use `exiftool` to verify metadata on anything before it leaves your hands. This is basic OPSEC but people skip it constantly.

### The Noisy Nmap

Day one on the network. Junior operator wanted to enumerate the network quickly. Ran:

```
nmap -sV -O 192.168.0.0/16
```

That is a version and OS detection scan against 65,536 IP addresses. Nmap sends thousands of packets per second. Every IDS and network monitoring tool on the planet has a signature for this exact traffic pattern.

The SOC saw 150,000 suspicious packets originating from one workstation in 4 minutes. They isolated the workstation immediately. The compromised machine, the initial foothold, gone. The engagement had to be restarted from the phishing phase.

**What this teaches you:** Recon on the internal network is not free. Every port scan, every service query, every LDAP enumeration leaves traces. You scan one target at a time. You scan slowly. You use targeted queries, not broadcast discovery. You think about what logs your action generates before you run it.

---

## The Full Infrastructure We Are Building

Here is exactly what we are building and why each component is here.

```
YOUR OPERATOR MACHINE (any OS, your workstation)
       |
       | SSH (port 22, key auth only)
       |
       v
+-------------------+         +--------------------+
|   C2 SERVER       |<--------|   REDIRECTOR VM    |<--- Victim machine
|   Azure VM        | port    |   Azure VM         |     (HTTPS port 443)
|   Ubuntu 22.04    | 8443    |   Ubuntu 22.04     |
|   Sliver C2       |  (HTTP) |   Nginx + SSL cert |
|                   |         |   Let's Encrypt    |
|   NO public port  |         |                    |
|   443 exposure    |         |   Port 443 open    |
|   Firewall: only  |         |   to internet      |
|   allows redirector|        |                    |
|   IP + your IP    |         +--------------------+
+-------------------+                  ^
                                        |
                                   DNS record points
                                   to this VM's IP
                                        |
                                +-------+--------+
                                |  Hostinger DNS  |
                                |  updates.domain.|
                                |  A record       |
                                +-----------------+
```

Let me walk you through every arrow in that diagram.

**Arrow 1: Victim machine to Redirector (HTTPS 443)**
The implant running on the victim machine makes an HTTPS request to your domain. DNS resolves that domain to your redirector's IP. Port 443 is open on the redirector. Nginx answers the connection, handles the TLS handshake using the Let's Encrypt certificate.

**Arrow 2: Redirector to C2 Server (HTTP 8443)**
Nginx, after receiving the implant's request, forwards it as plain HTTP to the C2 server on port 8443. This is an internal forwarding step. The connection between redirector and C2 server is inside Azure's virtual network, not going over the public internet. So plain HTTP between them is acceptable here (though you could add a layer of mTLS here too for high-security engagements).

**Arrow 3: You to C2 Server (SSH 22)**
You connect to the C2 server over SSH using your key pair. You open the Sliver console. You see the beacons. You run commands. The traffic goes direct from your operator machine to the C2 server's SSH port. Your IP is whitelisted at the firewall. Nobody else can SSH into it.

**What is NOT in this diagram:**
- Any direct path from the victim machine to the C2 server. The victim machine has no idea the C2 server exists.
- Any open port on the C2 server that faces the public internet (except SSH from your IP only).

This architecture means:
- If the redirector's IP gets blocked, you replace it in 10 minutes. C2 server untouched.
- If the domain gets flagged, you can configure a new domain and update DNS. C2 server untouched.
- If someone scans the C2 server's IP, they find nothing. The ports are closed to them.
- The blue team can investigate the redirector all day. They will find a normal Nginx web server with a real SSL certificate.

---

## The Specific Tools We Are Using and Why

### Azure VMs

We use Azure because the engagement team already has Azure credits set up for this client. In other engagements you might use AWS, GCP, DigitalOcean, or Vultr. The principles are the same.

For this engagement:
- C2 Server: Ubuntu 22.04 LTS, Standard B2s (2 vCPUs, 4 GB RAM). This is enough for Sliver with a small number of beacons.
- Redirector: Ubuntu 22.04 LTS, Standard B1s (1 vCPU, 1 GB RAM). Nginx is lightweight.

Do not use a powerful VM for no reason. Unnecessary compute means unnecessary cost and sometimes unnecessary attention (large VMs with traffic spikes can get noticed by cloud provider abuse monitoring).

### Hostinger for DNS

We control the domain through Hostinger's hPanel. We will add a subdomain DNS record (an A record) pointing to the redirector VM's public IP. When the implant connects to our subdomain, Hostinger's DNS resolves it to the redirector.

We use a subdomain (like `cdn.ourdomain.com` or `updates.ourdomain.com`) rather than the root domain because it looks more like a specific service endpoint, which is what real software uses for CDN and update servers.

### Sliver C2

We install Sliver on the C2 server. It runs as a systemd service so it survives reboots. It listens for beacon connections on port 8443 (internal only, forwarded from Nginx). We connect to the Sliver operator console over SSH and work from there.

### Nginx on the Redirector

Nginx listens on port 443 with the Let's Encrypt certificate. It forwards C2 traffic to Sliver. It serves a default page for anything else. No server-side scripting, no unnecessary modules, just a lean reverse proxy.

### Certbot (Let's Encrypt)

Certbot is the official Let's Encrypt client. We run it on the redirector to get a free, trusted SSL certificate for our domain. Let's Encrypt certificates are DV (Domain Validated) certificates. They verify only that you control the domain. They are trusted by every modern browser and operating system. They are identical in appearance to certificates used by legitimate websites.

---

## OPSEC Rules We Follow on Every Engagement

Not just this one. Every one.

### Infrastructure Rules

1. **C2 server has no public-facing ports except SSH from your IP.** Period. No exceptions. Sliver does not listen directly on any internet-facing port.

2. **Redirectors are disposable.** Do not store anything important on a redirector. It is a traffic forwarder. If it gets burned, you kill it and make a new one.

3. **Different VMs for different functions.** C2 server and redirector are separate. If you had a payload staging server, that is a third separate VM.

4. **SSH key authentication only.** Disable password authentication on every VM. Keys only. If your SSH key is compromised, that is a separate problem, but password auth is so much weaker that it is not worth keeping.

5. **Firewall by default-deny.** Azure NSG and UFW both set to deny all by default, then explicitly allow only what is needed.

6. **Do not reuse infrastructure between engagements.** A C2 server used on one engagement might have its IP logged in threat intelligence feeds by the time you start the next. Start fresh.

### Communication Rules

1. **Implants connect to domain names, not raw IPs.** If you change the redirector (burn and replace), you update DNS. If the implant had a hardcoded IP, you would have to redeploy it.

2. **Always HTTPS on the implant side.** Port 443. Real certificate. No exceptions.

3. **Beacon interval with jitter.** We set a baseline interval and a jitter percentage. For this engagement: 60 second interval, 30% jitter. This means check-ins happen every 42 to 78 seconds, randomly distributed.

4. **Test your infrastructure before you deploy anything to a real target.** We will do this in full on Day 3. The test must pass completely before we touch anything on the real client network.

### Operational Rules

1. **Document everything you do.** Every command you run, every action you take. Your team lead (me) needs to know for the report. The client needs to know what happened in the engagement. If something goes wrong, documentation is how you reconstruct what occurred.

2. **When in doubt, stop and ask.** I would rather you ask me a question that interrupts my coffee than have you accidentally delete something on a production server.

3. **Stay in scope.** The Rules of Engagement define exactly what systems we are allowed to touch. You read them before you touch anything.

4. **No personal machines.** All engagement work happens on the engagement VM we set up for you. Not your laptop, not your desktop at home. The engagement VM.

5. **Clean up after yourself.** When the engagement ends, we tear down the infrastructure, delete the VMs, wipe the domains. We do not leave C2 servers running after an engagement.

---

## Your Knowledge Check

Before tomorrow, you should be able to answer these without looking at notes. These are not trick questions. They are the foundation of everything we do.

**1. Why does an implant make outbound connections instead of waiting for inbound connections?**

Because corporate firewalls block inbound connections from the internet to internal machines. Outbound HTTPS on port 443 is allowed because employees need to browse the internet. The implant uses the same outbound path that normal web traffic uses.

**2. What is the difference between a beacon and a session in Sliver?**

A session maintains a persistent connection (constant, real-time). A beacon checks in periodically, runs queued commands, and disconnects. Beacons are quieter and harder to detect because there is no permanent connection to find.

**3. Why does the implant connect to a domain name instead of an IP address?**

Because if the redirector gets burned, you can replace it with a new VM and update the DNS record. The domain stays the same, the IP behind it changes. The implant reconnects through the new redirector automatically. If you hardcoded an IP, you would have to redeploy the implant on the victim machine.

**4. What does JARM fingerprinting detect?**

JARM fingerprints TLS server implementations. It sends specific TLS handshake packets and records how the server responds. Different software (Sliver, Nginx, Apache, IIS) responds differently and has different JARM hashes. Defenders use JARM to identify what software is running on an IP, even without being able to decrypt the traffic.

**5. If the blue team blocks your redirector's IP, what do you do?**

Spin up a new redirector VM. Install Nginx, get a new certificate, configure the same proxy rules. Update the DNS A record for your domain to point to the new VM's IP. Wait for DNS propagation (usually a few minutes). Your C2 server and all your other infrastructure are untouched.

**6. Why is the C2 server not internet-facing?**

Because any internet-facing server can be scanned, fingerprinted, and identified. If your C2 server's port 443 is open to the internet, tools like Shodan, JARM scanners, and threat intel crawlers will find it and potentially identify it as a C2. We expose only the redirector. The C2 server accepts connections only from the redirector's IP and your operator IP.

---

## What We Build Over the Next Three Days

| Day | What We Do |
|-----|-----------|
| **Day 1** (today) | You understand the architecture, the OPSEC, and why every decision exists. This document. |
| **Day 2** | We spin up both Azure VMs, install Sliver on the C2 server, set up Nginx + Certbot on the redirector, configure the domain in Hostinger, firewall everything down tight, and test the full chain end-to-end. |
| **Day 3** | We generate a Sliver beacon configured to connect through the redirector, deploy it on our test Windows VM, verify the full traffic flow works, inspect it with Wireshark to see what the blue team sees, and confirm nothing leaks our real C2 server. |

After Day 3, if everything checks out, we are ready for the actual engagement.

One last thing: do not rush Day 2. I have seen operators set up infrastructure in 20 minutes and then spend 2 days troubleshooting why the beacon is not connecting. Take your time. Do each step, verify it works, then move to the next. The setup is not complicated but every step depends on the previous one being correct.

Let's go.

---

**Next Step:** [01_setup_sliver_c2.md](./01_setup_sliver_c2.md)

Setting up your Sliver C2 server and redirector on Azure, step by step.
