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

The client for this engagement is a financial institution called **Orsu Global Financial**. They have around 5,000 computers. They have a security team (called a SOC or blue team) that watches their network 24 hours a day.

To blend in with normal egress traffic, we do NOT use the client's name for our infrastructure. Instead, we registered an attacker domain on Hostinger named `ceruleanpay.online` to impersonate a third-party merchant payment gateway.

Our job is to simulate a real attacker. We try to get inside their network, move around, and reach their important systems without the blue team catching us.

Your job this week is to build the infrastructure that keeps us hidden while we do all of this.

---

## What is C2?

C2 stands for Command and Control.

When our malware (called an implant) runs on a computer inside the client's network, it needs to talk to us so we can send it instructions. C2 is that communication channel between us (outside) and the implant (inside).

**Why can't we just connect to the implant directly?**

Because the client's firewall blocks all incoming connections from the internet. Nothing from outside can reach inside.

However, the firewall does allow outgoing connections. Employees need to browse websites, use email, and download updates. So the firewall lets traffic go from inside to outside.

This means our implant has to start the connection. It reaches out from inside the network to our server on the internet. This is called a **reverse connection**.

**How the loop works:**

1. Implant runs on target computer.
2. Every 15 seconds (with random variation), implant connects to our server.
3. Implant asks: "Any commands for me?"
4. We send a command (example: "list all files in C:\Users").
5. Implant runs the command and sends us the output.
6. Implant disconnects and waits for the next check-in interval.
7. Loop repeats.

---

## The Wrong Way to Set Up C2

Here is how beginners do it and why it fails immediately:

```
Target Computer --> Internet --> Your C2 Server (Public IP: 20.120.45.12)
```

You rent a VM on Azure, install Sliver (our C2 tool), and tell the implant to connect directly to your server's IP address.

**What happens on a real engagement:**

The blue team's monitoring system (called a SIEM) collects logs of every connection target computers make to the internet. An analyst sees a computer connecting to your IP address every 15 seconds. 

The analyst visits your IP in her browser, sees a raw server or suspicious certificate, and blocks your IP at the perimeter firewall.

Because your real server IP was hardcoded inside the implant, your C2 server is now burned forever. You cannot use it for the rest of the engagement.

---

## The Right Way: The Domain Redirector

We put a separate server running Nginx between the target and our real C2 server.

```
Target Computer --> Domain Redirector (Nginx + SSL) --> C2 Server (Sliver)
```

The middle server is called a **redirector**. Its only job is to receive connections from implants and pass them to the real C2 server.

**Why this protects us:**

1. **The C2 server is invisible**: The implant only knows the redirector's domain (`updates.ceruleanpay.online`). The real C2 server has no direct public exposure and only accepts traffic from the redirector.
2. **Filtering Scanners**: Nginx checks the request URL path and headers. If the request goes to our secret path (`/updates/v2/sync`), Nginx forwards it to Sliver on port 8443.
3. **Decoy Landing Page**: If a blue team analyst or automated scanner visits `updates.ceruleanpay.online` in a browser, Nginx serves a fake merchant landing page (`CeruleanPay Gateway`). They see a normal-looking page and move on.
4. **Disposable Infrastructure**: If the blue team blocks our redirector IP, we destroy the redirector VM and spin up a new one in minutes. The C2 server stays online and untouched.

---

## The Complete Architecture We Are Building

```
[ WINDOWS TARGET VM ]
       |
       | HTTPS (Port 443) -> updates.ceruleanpay.online/updates/v2/sync
       v
[ NGINX REDIRECTOR VM ] (Azure VM - Public IP)
       | (Filters traffic, terminates Let's Encrypt SSL, serves Decoy site to scanners)
       | Proxy Pass HTTPS (Port 8443)
       v
[ C2 SERVER VM ] (Azure VM - Private Subnet behind NAT Gateway)
       | (Runs Sliver C2, HTTPS Listener on :8443, direct SSH console)
       v
[ OPERATOR TERMINAL ] (SSH access to C2 Server)
```

---

## OPSEC Rules

1. **The C2 server is never exposed directly**: Port 8443 on the C2 server is locked down at both the Azure NSG and UFW firewall levels to accept traffic ONLY from the redirector VM.
2. **Use a secret URI path**: Never use default C2 paths. We route implant traffic through a custom path (`/updates/v2/sync`).
3. **The redirector is disposable**: Never store sensitive files, credentials, or logs on the redirector.
4. **Test everything before engagement**: Verify target `netstat` telemetry shows only the redirector IP, not the C2 IP.

---

## Your 4-Day Plan

* **Day 1: The Briefing** (This Document) - Architecture, OPSEC concepts, and flow.
* **Day 2: C2 Server Setup** - Provision Azure `c2-server` VM, configure Azure NAT Gateway for private subnets, harden with `fail2ban` and UFW, install Sliver, start HTTPS listener on port 8443.
* **Day 3: Domain & Redirector Setup** - Configure Hostinger DNS (`updates.ceruleanpay.online`), provision Azure `redirector` VM, install Nginx + Certbot Let's Encrypt SSL, build merchant decoy site, configure reverse proxy with secret path routing and scanner blocking.
* **Day 4: Payload Generation & Verification** - Generate stageless Windows EXE beacon in Sliver, execute on Windows Target VM, verify session, inspect `netstat` and Nginx access logs.

---

## Tools & Concepts Summary

| Tool / Concept | Purpose | Location / Scope |
|---|---|---|
| **Sliver** | Open-source C2 framework | C2 Server VM |
| **Nginx** | Reverse proxy, SSL termination, scanner filtering, decoy site | Redirector VM |
| **Certbot** | Free Let's Encrypt TLS certificate manager | Redirector VM |
| **Azure NAT Gateway** | Provides stable outbound internet for private VNet subnets | Azure Virtual Network |
| **Azure NSG & UFW** | Network & host firewalls (port 22 locked to operator, 8443 locked to redirector) | Azure & Linux VMs |
| **Hostinger** | Domain registrar & DNS management (`updates.ceruleanpay.online`) | Cloud DNS |

---
**Next Step**: Proceed to [01_setup_sliver_c2.md](01_setup_sliver_c2.md) to set up your C2 server on Azure.
