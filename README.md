# Mosquitto on Raspberry Pi: remote access via port forwarding (port 8883) + TLS, plus Node-RED

Guide to set up a Mosquitto broker with username/password authentication and TLS, reachable from the internet for the iOS app **Tru-Control** (controls a Truma Combi heater through an MQTT/BLE gateway), by opening port 8883 on the router. The TLS certificate is issued by Let's Encrypt (a public CA), so it works with a "standard profile" client — no custom certificate or CA needs to be installed on the app.

You already have a domain managed on Cloudflare: we'll only use it as DNS (no proxy/tunnel) to give your home's public IP a stable hostname, and to obtain the certificate automatically.

## Overview

1. Mosquitto: install, create user/password, TLS listener on 8883.
2. Router: reserved/static IP for the Pi + port forwarding 8883 → Pi.
3. DNS on Cloudflare: a record pointing to your public IP (no proxy) + optional dynamic update if your home IP changes.
4. Let's Encrypt certificate via DNS-01 (Cloudflare plugin), automatic renewal.
5. Firewall/hardening, since the broker is now genuinely exposed to the internet.
6. Testing and configuring Tru-Control.
7. Node-RED: installation and local-network access (port 1880 on the local firewall).

## Part 1 — Base Mosquitto installation and configuration

Run on the Raspberry Pi over SSH.

```bash
sudo apt update
sudo apt install -y mosquitto mosquitto-clients
sudo systemctl enable mosquitto
```

### Create the user with a password

```bash
sudo mosquitto_passwd -c /etc/mosquitto/passwd your_username
# you'll be asked for the password twice
```

To add more users later (without `-c`, which overwrites the file):

```bash
sudo mosquitto_passwd /etc/mosquitto/passwd another_username
```

### Password file permissions (important step, common cause of crashes)

The Mosquitto broker runs as the system user `mosquitto`, not as root. The command above creates the file as `root:root` with `600` permissions, so the broker can't read it and the service dies immediately on startup (in `journalctl` you'll see `mosquitto.service: Main process exited, code=exited, status=13`). Fix the permissions right after creating it:

```bash
sudo chown root:mosquitto /etc/mosquitto/passwd
sudo chmod 640 /etc/mosquitto/passwd
```

Repeat this every time you add a user with `mosquitto_passwd` (the command rewrites the file and resets its permissions to `600 root:root`).

We'll add the actual TLS listener in Part 4, after obtaining the certificate (Mosquitto won't start if the config file points to certificate files that don't exist yet).

## Part 2 — Router: reserved IP + port forwarding

### 1. Reserve a fixed IP for the Raspberry Pi

In your router's admin panel, look for the DHCP section (often called "DHCP reservation" or "static IP") and assign a fixed IP to the Pi based on its MAC address (find it with `ip link show` on the Pi). This way the port forwarding rule won't break after the next reboot/DHCP renewal.

### 2. Open port 8883

In the router's "Port Forwarding" / "NAT" / "Virtual Server" section, create a rule:

- Protocol: TCP
- External port: 8883
- Internal IP: the Pi's fixed IP assigned above
- Internal port: 8883

Nothing else needs to be opened (no 1883, no 80): no plaintext traffic exposed, and we obtain the certificate without needing port 80 (see Part 4).

## Part 3 — DNS on Cloudflare

You need a stable hostname pointing to your home's public IP, e.g. `mqtt.yourdomain.com`.

### 1. Create the DNS record

In Cloudflare's DNS panel, add a record:

- Type: `A`
- Name: `mqtt`
- Content: your current public IP (check with `curl ifconfig.me` on the Pi)
- **Proxy status: DNS only (grey cloud, not orange)**

This last point matters: if you leave the Cloudflare proxy on (orange cloud), plain-TCP MQTT traffic won't get through — the Cloudflare proxy only forwards HTTP(S). With "DNS only" the record points straight to your home IP, like a normal DNS entry.

### 2. If your public IP isn't static (the common case for residential lines)

You need to update the record whenever the IP changes. The simplest approach is a small cron script on the Pi that uses the Cloudflare API:

1. Create an **API Token** on Cloudflare (not the Global API Key) with `Zone → DNS → Edit` permission, scoped only to your domain's zone.
2. Save the token, zone ID, and record ID in a script like:

```bash
#!/bin/bash
# /usr/local/bin/update-cloudflare-dns.sh
CF_API_TOKEN="your_token"
ZONE_ID="your_zone_id"
RECORD_ID="the_mqtt_A_record_id"
RECORD_NAME="mqtt.yourdomain.com"

CURRENT_IP=$(curl -s https://ifconfig.me)

curl -s -X PUT "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records/$RECORD_ID" \
  -H "Authorization: Bearer $CF_API_TOKEN" \
  -H "Content-Type: application/json" \
  --data "{\"type\":\"A\",\"name\":\"$RECORD_NAME\",\"content\":\"$CURRENT_IP\",\"proxied\":false}"
```

```bash
sudo chmod +x /usr/local/bin/update-cloudflare-dns.sh
sudo crontab -e
# add:
*/15 * * * * /usr/local/bin/update-cloudflare-dns.sh >/dev/null 2>&1
```

If your provider gives you a static IP (ask them if you're not sure), you can skip this: just set the record once.

## Part 4 — Public TLS certificate with Let's Encrypt (DNS-01, no need to open port 80)

We use the DNS-01 challenge with certbot's Cloudflare plugin: it doesn't require any extra open port, works even if your IP changes, and gives you a certificate trusted by any standard client.

### 1. Install certbot with the Cloudflare plugin

```bash
sudo apt install -y certbot python3-certbot-dns-cloudflare
```

### 2. Create credentials for the plugin

```bash
sudo mkdir -p /etc/letsencrypt
sudo nano /etc/letsencrypt/cloudflare.ini
```

Content (use the same API Token created above, with `Zone:DNS:Edit` permission on the domain's zone):

```ini
dns_cloudflare_api_token = your_token
```

```bash
sudo chmod 600 /etc/letsencrypt/cloudflare.ini
```

### 3. Request the certificate

```bash
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini \
  -d mqtt.yourdomain.com
```

The certificate ends up in `/etc/letsencrypt/live/mqtt.yourdomain.com/`.

### 4. Make the certificates readable by Mosquitto and automate renewal

Let's Encrypt's files are readable only by root; Mosquitto runs as its own dedicated user. The cleanest approach is to copy them into a folder Mosquitto can read every time they're renewed, via a certbot "deploy hook".

```bash
sudo mkdir -p /etc/mosquitto/certs
sudo nano /etc/letsencrypt/renewal-hooks/deploy/mosquitto-reload.sh
```

Content:

```bash
#!/bin/bash
cp /etc/letsencrypt/live/mqtt.yourdomain.com/fullchain.pem /etc/mosquitto/certs/
cp /etc/letsencrypt/live/mqtt.yourdomain.com/privkey.pem /etc/mosquitto/certs/
chown mosquitto:mosquitto /etc/mosquitto/certs/*.pem
chmod 640 /etc/mosquitto/certs/*.pem
systemctl restart mosquitto
```

```bash
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/mosquitto-reload.sh
# run it once by hand to populate /etc/mosquitto/certs
sudo /etc/letsencrypt/renewal-hooks/deploy/mosquitto-reload.sh
```

Certbot already installs a systemd timer (`certbot.timer`) that checks for renewal twice a day and only renews when needed (Let's Encrypt certificates last 90 days); the script above runs automatically on every renewal, so from here on it's all automatic. You can verify the timer is active with:

```bash
systemctl status certbot.timer
```

## Part 5 — TLS listener on Mosquitto

Create/update `/etc/mosquitto/conf.d/remote.conf`:

```conf
# IMPORTANT: recent versions of Mosquitto (the default on Raspberry Pi OS)
# have "per_listener_settings true" in /etc/mosquitto/mosquitto.conf. With
# this option on, allow_anonymous/password_file written BEFORE a "listener"
# line only apply to an implicit default listener, not to the listeners
# declared explicitly below. They must therefore be repeated after every
# "listener" line, otherwise that listener has no authentication backend at
# all and rejects everyone with "not authorised" (CONNACK 5), no matter how
# correct the credentials are.

# Public TLS listener, the one exposed via the router's port forwarding
listener 8883
certfile /etc/mosquitto/certs/fullchain.pem
keyfile /etc/mosquitto/certs/privkey.pem
require_certificate false
allow_anonymous false
password_file /etc/mosquitto/passwd

# Plaintext listener ONLY for local loopback testing (not exposed to the
# internet, not forwarded by the router). Deliberately without
# password_file: it stays unusable by anyone (no anonymous access, no
# authentication backend), since it's not needed from the outside.
listener 1883 127.0.0.1
allow_anonymous false
```

`require_certificate false` matters: it means Mosquitto requires server-side TLS (the client verifies the server's certificate) but does not demand a client certificate — consistent with your requirement of a "standard profile", no client-side certificate.

Restart:

```bash
sudo systemctl restart mosquitto
sudo systemctl status mosquitto
```

### If the service won't start (exit status 13)

`status=13` almost always means "permission denied" reading one of the files referenced in the config (`password_file`, `certfile`, or `keyfile`) — the broker runs as the `mosquitto` user, not root. Diagnose with:

```bash
sudo journalctl -u mosquitto -n 30 --no-pager
ls -l /etc/mosquitto/passwd
ls -l /etc/mosquitto/certs/
```

The line logged directly by Mosquitto (not by systemd) tells you which file is the problem. Fix the permissions of the file involved:

```bash
sudo chown root:mosquitto /etc/mosquitto/passwd
sudo chmod 640 /etc/mosquitto/passwd

sudo chown mosquitto:mosquitto /etc/mosquitto/certs/*.pem
sudo chmod 640 /etc/mosquitto/certs/*.pem
```

Then restart again with `sudo systemctl restart mosquitto`.

## Part 6 — Firewall and hardening

With port 8883 genuinely exposed to the internet, this step matters more than it would with a tunnel.

```bash
sudo apt install -y ufw
sudo ufw allow 22/tcp        # SSH, if you use it — be careful not to lock yourself out
sudo ufw allow 8883/tcp
sudo ufw enable
```

Additional recommendations:

- Long, random MQTT passwords, unique per user/device.
- Consider `fail2ban` with a filter on the Mosquitto log to block IPs that repeatedly fail authentication (exposed MQTT brokers get scanned regularly by bots on the internet).
- If you'll have multiple users/devices with different permissions in the future, use an ACL file (`acl_file`) to restrict topics per user.
- Keep Raspberry Pi OS and Mosquitto updated (`sudo apt update && sudo apt upgrade`).
- Check `/var/log/mosquitto/mosquitto.log` from time to time for suspicious access attempts.

## Part 7 — Testing and configuring Tru-Control

### Test from outside your home network (e.g. on mobile data, not your home Wi-Fi)

```bash
mosquitto_pub -h mqtt.yourdomain.com -p 8883 \
  --tls-use-os-certs \
  -u your_username -P your_password \
  -t test/topic -m "hello"
```

If it doesn't return certificate or connection errors, the end-to-end path works.

### Configure the app

In Tru-Control's MQTT settings:

- **Host**: `mqtt.yourdomain.com`
- **Port**: `8883`
- **TLS/SSL**: on, "standard" profile/certificate (no certificate or CA to import)
- **Username / Password**: the ones created with `mosquitto_passwd`

## Part 8 — Node-RED: installation and local-network access

Node-RED is handy for bridging/automating flows around the broker (e.g. relaying data between the gateway and other services). This section covers installing it on the Pi and reaching its editor from a browser on your local network — it does **not** cover exposing it to the internet, for the security reason explained below.

### 1. Install Node-RED

```bash
bash <(curl -sL https://raw.githubusercontent.com/node-red/linux-installers/master/deb/update-nodejs-and-nodered)
```

The official installer also takes care of installing/updating Node.js if needed. Follow the on-screen prompts (you can accept the defaults).

### 2. Enable it as a service and start it

```bash
sudo systemctl enable nodered.service
sudo systemctl start nodered.service
sudo systemctl status nodered.service
```

### 3. Open port 1880 on the local firewall

If you followed Part 6, `ufw` is active and only allows 22 (SSH) and 8883 (MQTT) — that's why the Node-RED editor doesn't load in the browser. Add a rule for Node-RED's default port:

```bash
sudo ufw allow 1880/tcp
sudo ufw reload
sudo ufw status
```

Then open `http://<Pi-local-IP>:1880` from a browser on a device connected to the same home Wi-Fi (use the fixed IP you reserved for the Pi in Part 2).

### Important security note

**Do not forward port 1880 on the router.** Unlike Mosquitto, Node-RED's editor has no login by default — anyone who could reach it over the internet would be able to run arbitrary code on the Pi. Keep it reachable only from your LAN, as set up above.

If you ever need remote access to Node-RED, set up its built-in authentication first (`adminAuth` in `~/.node-red/settings.js`) before considering any kind of external exposure — ask if you'd like that section added.

## Appendix — Alternatives without opening ports (if you change your mind later)

If in the future you'd rather not keep a port permanently open on the router, two alternatives were already validated while researching this guide:

- **Tailscale Funnel**: TCP+TLS with an automatic public certificate, no port to open, compatible with plain-TCP MQTT (likely Tru-Control's case). Simpler to turn on/off in parallel with the port-forwarding setup.
- **Cloudflare Tunnel**: requires the client to support MQTT over WebSocket (`wss://`), which wasn't confirmed for Tru-Control; tunneling plain-TCP MQTT through Cloudflare is only available with Spectrum (a paid Enterprise plan).

Let me know if you'd like the detailed instructions for either path added back in.
