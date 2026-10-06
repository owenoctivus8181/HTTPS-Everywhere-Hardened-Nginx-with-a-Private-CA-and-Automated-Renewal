# HTTPS Everywhere: Hardened Nginx with a Private CA and Automated Renewal

A hands-on lab that secures a web server with HTTPS using **Nginx**, a **private Certificate Authority (CA)** built with `openssl`, an **HTTP→HTTPS redirect**, **HSTS**, **TLS 1.2/1.3-only hardening**, and **automatic certificate renewal** with a systemd timer. The result is verified with `testssl.sh`.

> **Note on Let's Encrypt:** This lab runs locally without a public domain, and Let's Encrypt can only issue certificates to servers it can reach from the internet. So the lab uses its own CA, which follows the same workflow (key → request → signed certificate → renewal). The Nginx hardening is identical to a production Let's Encrypt setup. See [Going to production](#going-to-production-with-lets-encrypt) for the Certbot version.

## Architecture

```
                 ┌────────────────────────────┐
                 │  Lab Local CA (openssl)    │
                 │  ca.key / ca.crt           │
                 └─────────────┬──────────────┘
                               │ signs
                      ┌────────▼────────┐
                      │ lab.crt/lab.key │  (90-day certificate)
                      └────────┬────────┘
                               │
Client ──HTTP :80──► Nginx ──301──► HTTPS :443 ──► site
(curl/browser)       (redirect)     (TLS 1.2/1.3, HSTS)

systemd timer (daily) ──► renew.sh ──► new cert ──► nginx reload
```

## What this project demonstrates

| Feature | How |
|---|---|
| Certificate lifecycle | Private CA, CSR, signed server certificate with SAN |
| HTTP → HTTPS redirect | `return 301 https://$host$request_uri;` |
| HSTS | `Strict-Transport-Security` header |
| TLS hardening | TLS 1.2/1.3 only, ECDHE ciphers with forward secrecy, session tickets off |
| Automated renewal | Script plus systemd timer that renews within 30 days of expiry |
| Verification | `curl`, `openssl s_client`, and `testssl.sh` |

## Prerequisites

- Fedora or RHEL-family Linux with `sudo` (SELinux enforcing; commands include `restorecon`)
- `nginx`, `openssl`, `curl`, `git`
- Ports 80 and 443 free on the machine

```bash
sudo dnf install -y nginx openssl curl git
```

## Step-by-step setup

### Step 1: Map the lab name to your machine

```bash
echo "127.0.0.1 lab.local www.lab.local" | sudo tee -a /etc/hosts
getent hosts lab.local
```

### Step 2: Create the private CA

```bash
mkdir -p ~/tls-lab && cd ~/tls-lab

openssl genrsa -out ca.key 4096
openssl req -x509 -new -key ca.key -sha256 -days 3650 \
  -subj "/CN=Lab Local CA" -out ca.crt
```

`ca.crt` is the public certificate of the CA. `ca.key` is its secret key and must never be shared.

### Step 3: Issue the server certificate

```bash
openssl genrsa -out lab.key 2048
openssl req -new -key lab.key -subj "/CN=lab.local" -out lab.csr

cat > lab.ext <<'EOF'
subjectAltName=DNS:lab.local,DNS:www.lab.local
basicConstraints=CA:FALSE
keyUsage=digitalSignature
extendedKeyUsage=serverAuth
EOF

openssl x509 -req -in lab.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out lab.crt -days 90 -sha256 -extfile lab.ext
```

Why each part matters:

| Item | Purpose |
|---|---|
| `lab.key` | The server's private key |
| `lab.csr` | A request asking the CA to sign the key |
| `subjectAltName` | Modern clients ignore the CN and check the SAN list |
| `-days 90` | Matches Let's Encrypt's lifetime, so renewal is meaningful |

Verify the result:

```bash
openssl x509 -in lab.crt -noout -subject -dates -ext subjectAltName
```

### Step 4: Install the certificate for Nginx

```bash
sudo mkdir -p /etc/nginx/tls
sudo cp lab.crt lab.key /etc/nginx/tls/
sudo chmod 600 /etc/nginx/tls/lab.key
sudo restorecon -Rv /etc/nginx/tls
```

### Step 5: Configure Nginx

Create `/etc/nginx/conf.d/lab.conf`:

```nginx
# Port 80: redirect everything to HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name lab.local www.lab.local;
    return 301 https://$host$request_uri;
}

# Port 443: the secured site
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name lab.local www.lab.local;
    root /usr/share/nginx/html;

    ssl_certificate     /etc/nginx/tls/lab.crt;
    ssl_certificate_key /etc/nginx/tls/lab.key;

    # Modern protocols only
    ssl_protocols TLSv1.2 TLSv1.3;

    # Strong ciphers with forward secrecy (applies to TLS 1.2)
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;
    ssl_prefer_server_ciphers off;

    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;

    # HSTS (short while testing, see "HSTS rollout")
    add_header Strict-Transport-Security "max-age=300" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
}
```

| Setting | Meaning |
|---|---|
| `return 301 ...` | Permanent redirect from HTTP to HTTPS |
| `ssl_protocols TLSv1.2 TLSv1.3` | Disables SSLv3 and TLS 1.0/1.1 |
| `ssl_ciphers` | Only ECDHE (forward secrecy) with AES-GCM or ChaCha20 |
| `ssl_session_tickets off` | Improves forward secrecy |
| `Strict-Transport-Security` | Tells browsers to use HTTPS only for this site |

Start Nginx and open the firewall:

```bash
sudo nginx -t
sudo systemctl enable --now nginx
sudo firewall-cmd --permanent --add-service={http,https}
sudo firewall-cmd --reload
```

If `nginx -t` rejects `http2 on;`, use `listen 443 ssl http2;` on older Nginx versions.

### Step 6: Trust the CA

The system does not trust your CA yet, which is why a plain `curl https://lab.local` fails. Add it to the system trust store:

```bash
sudo cp ~/tls-lab/ca.crt /etc/pki/ca-trust/source/anchors/lab-ca.crt
sudo update-ca-trust
```

For Firefox, import `ca.crt` under Settings → Privacy & Security → Certificates → View Certificates → Authorities.

## Verification

### Redirect and HSTS

```bash
curl -I http://lab.local       # expect: 301 and Location: https://lab.local/
curl -I https://lab.local      # expect: 200 and Strict-Transport-Security header
```

### TLS versions and ciphers

```bash
openssl s_client -connect lab.local:443 -tls1_2 </dev/null | grep -E "Protocol|Cipher"
openssl s_client -connect lab.local:443 -tls1_3 </dev/null | grep -E "Protocol|Cipher"
```

### testssl.sh

```bash
git clone --depth 1 https://github.com/drwetter/testssl.sh.git
cd testssl.sh
./testssl.sh --add-ca ~/tls-lab/ca.crt https://lab.local
```

Expected findings:

- SSLv2, SSLv3, TLS 1.0, TLS 1.1: **not offered**
- TLS 1.2 and TLS 1.3: **offered**
- Only forward-secrecy ciphers
- HSTS present

### Results

| Check | Result |
|---|---|
| HTTP redirects to HTTPS | [PASS/FAIL, paste evidence] |
| TLS 1.0/1.1 disabled | [PASS/FAIL] |
| TLS 1.2 and 1.3 enabled | [PASS/FAIL] |
| HSTS header present | [PASS/FAIL] |
| testssl.sh findings | [summary, e.g. no high/critical issues] |
| Renewal tested | [PASS/FAIL] |

## Automated renewal

`renew.sh` renews the certificate only if it expires within 30 days, then reloads Nginx.

```bash
#!/bin/bash
set -euo pipefail
cd /home/YOUR_USERNAME/tls-lab

# Renew only if the cert expires within 30 days (2592000 seconds)
if openssl x509 -checkend 2592000 -noout -in /etc/nginx/tls/lab.crt; then
    echo "Certificate still valid for more than 30 days, nothing to do."
    exit 0
fi

echo "Renewing certificate..."
openssl req -new -key lab.key -subj "/CN=lab.local" -out lab.csr
openssl x509 -req -in lab.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out lab.crt -days 90 -sha256 -extfile lab.ext

cp lab.crt /etc/nginx/tls/lab.crt
restorecon /etc/nginx/tls/lab.crt
nginx -t && systemctl reload nginx
echo "Renewed and Nginx reloaded."
```

Replace `YOUR_USERNAME` with your real username, then install it:

```bash
sudo cp renew.sh /usr/local/sbin/renew-lab-cert.sh
sudo chmod +x /usr/local/sbin/renew-lab-cert.sh
```

Schedule it with a systemd timer:

```bash
sudo tee /etc/systemd/system/renew-lab-cert.service <<'EOF'
[Unit]
Description=Renew lab TLS certificate

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/renew-lab-cert.sh
EOF

sudo tee /etc/systemd/system/renew-lab-cert.timer <<'EOF'
[Unit]
Description=Daily check for lab TLS certificate renewal

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now renew-lab-cert.timer
systemctl list-timers | grep renew-lab
```

### Testing a real renewal

A fresh certificate is not near expiry, so the script does nothing. To force a renewal, issue a short-lived certificate (use `-days 10` in Step 3), copy it to `/etc/nginx/tls/`, run `sudo /usr/local/sbin/renew-lab-cert.sh`, and compare the dates before and after:

```bash
openssl x509 -in /etc/nginx/tls/lab.crt -noout -dates
```

## HSTS rollout

HSTS is cached by browsers, so a wrong value is hard to undo. Increase it in stages:

| Stage | Header value |
|---|---|
| Testing | `max-age=300` (5 minutes) |
| After everything works | `max-age=31536000` (1 year) |
| Only with full understanding | add `includeSubDomains` and `preload` |

## Troubleshooting (issues hit while building this)

| Symptom | Cause | Fix |
|---|---|---|
| `cannot load certificate "/etc/nginx/tls/lab.crt" ... No such file` | Certificate was not created or not copied to `/etc/nginx/tls` | Re-run Step 3 and Step 4, then `sudo ls -l /etc/nginx/tls` |
| `curl: (7) Failed to connect to lab.local port 443` | Nginx not running or HTTPS block not loaded | `sudo systemctl status nginx`, `sudo ss -tlnp \| grep :443`, `sudo nginx -T \| grep "listen 443"` |
| Nginx fails with `Address already in use` | Another service (Apache, a container) holds port 80 | `sudo ss -tlnp \| grep ':80 '`, stop the other service |
| `curl: (60) SSL certificate problem` | System does not trust the private CA | Step 6, or use `curl --cacert ca.crt` |
| `key values mismatch` | Certificate and key do not belong together | Regenerate both, copy both |
| Old config conflicts | Leftover files in `/etc/nginx/conf.d/` | Move old `.conf` files away and run `nginx -t` |

## Security notes

- **Never commit private keys.** Add this to `.gitignore`:

```
  *.key
  *.csr
  *.srl
  ca.crt
```

- A private CA is only trusted where you install it. Never ship `ca.key` or install this CA on machines you do not control.
- This lab uses RSA keys for simplicity. ECDSA (`openssl ecparam`) is a good next experiment.

## Going to production with Let's Encrypt

With a real domain and a public server (for example a cloud VM), the setup is almost identical:

1. Point a DNS A record at the server and open ports 80 and 443.
2. Install Certbot: `sudo dnf install -y certbot python3-certbot-nginx`
3. Get the certificate (test with `--staging` first):

```bash
   sudo certbot certonly --nginx -d example.com -d www.example.com
```

4. Change `ssl_certificate` and `ssl_certificate_key` in the Nginx config to `/etc/letsencrypt/live/example.com/fullchain.pem` and `privkey.pem`.
5. Enable renewal: `sudo systemctl enable --now certbot-renew.timer`, and add a deploy hook that runs `systemctl reload nginx`.
6. Test renewal with `sudo certbot renew --dry-run`.
7. Scan the public site with [SSL Labs](https://www.ssllabs.com/ssltest/). Expect an A grade, or A+ once HSTS is set to at least six months.

## Cleanup

```bash
sudo systemctl disable --now renew-lab-cert.timer
sudo rm /etc/nginx/conf.d/lab.conf
sudo rm /etc/pki/ca-trust/source/anchors/lab-ca.crt && sudo update-ca-trust
sudo sed -i '/lab.local/d' /etc/hosts
```

## Skills demonstrated

PKI fundamentals (CA, CSR, SAN, certificate chains), TLS hardening, HSTS, automated certificate lifecycle management, systemd timers, SELinux-aware file handling, and security verification with `testssl.sh`.
