# Mosquitto su Raspberry Pi: accesso remoto con port forwarding (porta 8883) + TLS

Guida per configurare un broker Mosquitto con autenticazione username/password e TLS, raggiungibile da internet per l'app iOS **Tru-Control** (pilota un riscaldatore Truma Combi via un gateway MQTT/BLE), aprendo la porta 8883 sul router. Il certificato TLS viene emesso da Let's Encrypt (CA pubblica), quindi resta valido per un client "a profilo standard" — nessun certificato o CA custom da installare sull'app.

Hai già un dominio gestito su Cloudflare: lo useremo solo come DNS (senza proxy/tunnel) per assegnare un hostname stabile al tuo IP di casa e per ottenere il certificato in modo automatico.

## Panoramica dei passaggi

1. Mosquitto: installazione, utente/password, listener TLS su 8883.
2. Router: IP statico/riservato per il Pi + port forwarding 8883 → Pi.
3. DNS su Cloudflare: record per il tuo IP pubblico (senza proxy) + eventuale aggiornamento dinamico se l'IP di casa cambia.
4. Certificato Let's Encrypt via DNS-01 (plugin Cloudflare), rinnovo automatico.
5. Firewall/hardening, dato che ora il broker è davvero esposto su internet.
6. Test e configurazione di Tru-Control.

## Parte 1 — Installazione e configurazione base di Mosquitto

Da eseguire sul Raspberry Pi via SSH.

```bash
sudo apt update
sudo apt install -y mosquitto mosquitto-clients
sudo systemctl enable mosquitto
```

### Crea l'utente con password

```bash
sudo mosquitto_passwd -c /etc/mosquitto/passwd nome_utente
# ti chiede la password due volte
```

Per aggiungere altri utenti in seguito (senza `-c`, che sovrascrive il file):

```bash
sudo mosquitto_passwd /etc/mosquitto/passwd altro_utente
```

Il listener TLS vero e proprio lo aggiungiamo nella Parte 4, dopo aver ottenuto il certificato (Mosquitto non parte se il file di configurazione punta a certificati che non esistono ancora).

## Parte 2 — Router: IP riservato + port forwarding

### 1. Riserva un IP fisso al Raspberry Pi

Nel pannello di amministrazione del router, cerca la sezione DHCP (spesso "DHCP reservation" o "IP statico") e assegna un IP fisso al Pi in base al suo indirizzo MAC (lo trovi con `ip link show` sul Pi). Così il port forwarding non si rompe al prossimo riavvio/rinnovo DHCP.

### 2. Apri la porta 8883

Nella sezione "Port Forwarding" / "NAT" / "Virtual Server" del router, crea una regola:

- Protocollo: TCP
- Porta esterna: 8883
- IP interno: l'IP fisso del Pi assegnato sopra
- Porta interna: 8883

Non serve aprire nient'altro (niente 1883, niente 80): niente traffico in chiaro esposto, e il certificato lo otteniamo senza bisogno della porta 80 (vedi Parte 4).

## Parte 3 — DNS su Cloudflare

Ti serve un hostname stabile che punti al tuo IP pubblico di casa, es. `mqtt.tuodominio.com`.

### 1. Crea il record DNS

Nel pannello DNS di Cloudflare, aggiungi un record:

- Tipo: `A`
- Nome: `mqtt`
- Contenuto: il tuo IP pubblico attuale (verificabile con `curl ifconfig.me` dal Pi)
- **Proxy status: DNS only (nuvoletta grigia, non arancione)**

Questo ultimo punto è importante: se lasci il proxy Cloudflare attivo (nuvoletta arancione), il traffico MQTT su TCP puro non passa — il proxy Cloudflare inoltra solo HTTP(S). Con "DNS only" il record punta direttamente al tuo IP di casa, come un DNS normale.

### 2. Se il tuo IP pubblico non è statico (caso più comune per le linee residenziali)

Devi aggiornare il record ogni volta che l'IP cambia. Il modo più semplice è un piccolo script cron sul Pi che usa l'API di Cloudflare:

1. Crea un **API Token** su Cloudflare (non la Global API Key) con permesso `Zone → DNS → Edit` limitato solo alla zona del tuo dominio.
2. Salva token, zone ID e record ID in uno script tipo:

```bash
#!/bin/bash
# /usr/local/bin/update-cloudflare-dns.sh
CF_API_TOKEN="il_tuo_token"
ZONE_ID="il_tuo_zone_id"
RECORD_ID="l_id_del_record_A_mqtt"
RECORD_NAME="mqtt.tuodominio.com"

CURRENT_IP=$(curl -s https://ifconfig.me)

curl -s -X PUT "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records/$RECORD_ID" \
  -H "Authorization: Bearer $CF_API_TOKEN" \
  -H "Content-Type: application/json" \
  --data "{\"type\":\"A\",\"name\":\"$RECORD_NAME\",\"content\":\"$CURRENT_IP\",\"proxied\":false}"
```

```bash
sudo chmod +x /usr/local/bin/update-cloudflare-dns.sh
sudo crontab -e
# aggiungi:
*/15 * * * * /usr/local/bin/update-cloudflare-dns.sh >/dev/null 2>&1
```

Se invece il tuo provider ti dà un IP statico (chiedilo se non sei sicuro), puoi saltare questo punto: imposti il record una volta sola.

## Parte 4 — Certificato TLS pubblico con Let's Encrypt (DNS-01, senza aprire la porta 80)

Usiamo la sfida DNS-01 con il plugin Cloudflare di certbot: non richiede nessuna porta aperta in più, funziona anche se il tuo IP cambia, e ti dà un certificato riconosciuto da qualsiasi client standard.

### 1. Installa certbot con il plugin Cloudflare

```bash
sudo apt install -y certbot python3-certbot-dns-cloudflare
```

### 2. Crea le credenziali per il plugin

```bash
sudo mkdir -p /etc/letsencrypt
sudo nano /etc/letsencrypt/cloudflare.ini
```

Contenuto (usa lo stesso API Token creato sopra, con permesso `Zone:DNS:Edit` sulla zona del dominio):

```ini
dns_cloudflare_api_token = il_tuo_token
```

```bash
sudo chmod 600 /etc/letsencrypt/cloudflare.ini
```

### 3. Richiedi il certificato

```bash
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini \
  -d mqtt.tuodominio.com
```

Il certificato finisce in `/etc/letsencrypt/live/mqtt.tuodominio.com/`.

### 4. Rendi i certificati leggibili da Mosquitto e automatizza il rinnovo

I file di Let's Encrypt sono leggibili solo da root; Mosquitto gira con un utente dedicato. Il modo più pulito è copiarli in una cartella accessibile a Mosquitto ogni volta che vengono rinnovati, tramite un "deploy hook" di certbot.

```bash
sudo mkdir -p /etc/mosquitto/certs
sudo nano /etc/letsencrypt/renewal-hooks/deploy/mosquitto-reload.sh
```

Contenuto:

```bash
#!/bin/bash
cp /etc/letsencrypt/live/mqtt.tuodominio.com/fullchain.pem /etc/mosquitto/certs/
cp /etc/letsencrypt/live/mqtt.tuodominio.com/privkey.pem /etc/mosquitto/certs/
chown mosquitto:mosquitto /etc/mosquitto/certs/*.pem
chmod 640 /etc/mosquitto/certs/*.pem
systemctl restart mosquitto
```

```bash
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/mosquitto-reload.sh
# esegui una prima volta a mano per popolare /etc/mosquitto/certs
sudo /etc/letsencrypt/renewal-hooks/deploy/mosquitto-reload.sh
```

Certbot installa già un timer di sistema (`certbot.timer`) che controlla il rinnovo due volte al giorno e rinnova solo quando necessario (i certificati Let's Encrypt durano 90 giorni); lo script sopra viene lanciato automaticamente ad ogni rinnovo, quindi da qui in poi è tutto automatico. Puoi verificare che il timer sia attivo con:

```bash
systemctl status certbot.timer
```

## Parte 5 — Listener TLS su Mosquitto

Crea/aggiorna `/etc/mosquitto/conf.d/remote.conf`:

```conf
# Autenticazione obbligatoria: niente accesso anonimo
allow_anonymous false
password_file /etc/mosquitto/passwd

# Listener TLS pubblico, quello esposto tramite il port forwarding del router
listener 8883
certfile /etc/mosquitto/certs/fullchain.pem
keyfile /etc/mosquitto/certs/privkey.pem
require_certificate false

# Listener in chiaro SOLO per test locali da loopback (non esposto su internet,
# non inoltrato dal router)
listener 1883 127.0.0.1
```

`require_certificate false` è importante: significa che Mosquitto richiede TLS server-side (il client verifica il certificato del server) ma non pretende un certificato client — coerente con il vincolo che hai posto ("profilo standard", nessun certificato lato client).

Riavvia:

```bash
sudo systemctl restart mosquitto
sudo systemctl status mosquitto
```

## Parte 6 — Firewall e hardening

Con la porta 8883 davvero esposta su internet, questo passaggio conta più che con un tunnel.

```bash
sudo apt install -y ufw
sudo ufw allow 22/tcp        # SSH, se lo usi — occhio a non tagliarti fuori
sudo ufw allow 8883/tcp
sudo ufw enable
```

Consigli aggiuntivi:

- Password MQTT lunghe e casuali, uniche per ogni utente/dispositivo.
- Valuta `fail2ban` con un filtro sul log di Mosquitto per bloccare IP che falliscono ripetutamente l'autenticazione (i broker MQTT esposti vengono scansionati regolarmente da bot su internet).
- Se in futuro avrai più utenti/dispositivi con permessi diversi, usa un file ACL (`acl_file`) per limitare i topic per utente.
- Tieni aggiornati Raspberry Pi OS e Mosquitto (`sudo apt update && sudo apt upgrade`).
- Controlla ogni tanto `/var/log/mosquitto/mosquitto.log` per tentativi di accesso sospetti.

## Parte 7 — Test e configurazione di Tru-Control

### Test da fuori casa (es. con i dati mobili, non sul Wi-Fi di casa)

```bash
mosquitto_pub -h mqtt.tuodominio.com -p 8883 \
  --tls-use-os-certs \
  -u nome_utente -P la_tua_password \
  -t test/topic -m "ciao"
```

Se non dà errori di certificato o connessione, il percorso end-to-end funziona.

### Configura l'app

Nelle impostazioni MQTT di Tru-Control:

- **Host**: `mqtt.tuodominio.com`
- **Porta**: `8883`
- **TLS/SSL**: attivo, profilo/certificato "standard" (nessun certificato o CA da importare)
- **Username / Password**: quelli creati con `mosquitto_passwd`

## Appendice — Alternative senza aprire porte (se cambi idea in futuro)

Se in futuro preferissi non tenere una porta aperta permanentemente sul router, restano disponibili due alternative già validate durante la ricerca per questa guida:

- **Tailscale Funnel**: TCP+TLS con certificato pubblico automatico, nessuna porta da aprire, compatibile con MQTT su TCP puro (probabilmente il caso di Tru-Control). Percorso più semplice da attivare/disattivare in parallelo a quello con port forwarding.
- **Cloudflare Tunnel**: richiede che il client supporti MQTT su WebSocket (`wss://`), cosa non confermata per Tru-Control; il tunneling di MQTT su TCP puro via Cloudflare è disponibile solo con Spectrum (piano Enterprise a pagamento).

Se vuoi, posso riaggiungere le istruzioni dettagliate di uno dei due percorsi in un secondo momento.
