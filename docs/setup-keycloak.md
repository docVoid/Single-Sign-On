# Installation von Keycloak SSO auf Raspberry Pi 5 mit Cloudflare Tunnel

## 1. Ziel

Ein Keycloak-Server auf einem Raspberry Pi 5, erreichbar über eine externe Domain (`sso.pngrtz.com`), mit HTTPS über Cloudflare Tunnel.

## 2. Hardware & Software

- Raspberry Pi 5
- Betriebssystem: Raspberry Pi OS
- Docker & Docker Compose installiert

## 3. Cloudflare Tunnel Einrichtung

1. **Cloudflare Login auf Raspberry Pi**

```bash
cloudflared tunnel login
```

- Browser öffnet Cloudflare Login
- Zertifikat wird lokal gespeichert: `/home/jp/.cloudflared/cert.pem`

2. **Tunnel erstellen**

```bash
cloudflared tunnel create keycloak
```

- Tunnel-ID wird angezeigt: `---`

3. **Tunnel-DNS-Route erstellen**

```bash
cloudflared tunnel route dns keycloak sso.pngrtz.com
```

- DNS-Eintrag (CNAME) wird bei Cloudflare gesetzt

4. **Tunnel-Konfig speichern**

- Datei: `/etc/cloudflared/config.yml`

```yaml
tunnel: keycloak
credentials-file: /root/.cloudflared/---.json

ingress:
  - hostname: sso.pngrtz.com
    service: http://localhost:8080
  - service: http_status:404
```

5. **Tunnel als Service starten**

```bash
sudo cloudflared service install
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
systemctl status cloudflared
```

- Status zeigt `active (running)`

## 4. Docker / Keycloak Setup

### 4.1 Docker Compose

Pfad: `~/keycloak/docker-compose.yml`

```yaml
services:
  postgres:
    image: postgres:16
    container_name: keycloak-db
    restart: unless-stopped
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: keycloak
      POSTGRES_PASSWORD: keycloakpass
    volumes:
      - ./postgres:/var/lib/postgresql/data

  keycloak:
    image: quay.io/keycloak/keycloak:26.0
    container_name: keycloak
    depends_on:
      - postgres
    restart: unless-stopped
    command: start
    environment:
      KC_DB: postgres
      KC_DB_URL_HOST: postgres
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD: keycloakpass

      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD:

      KC_PROXY: edge
      KC_HOSTNAME: sso.pngrtz.com
      KC_HOSTNAME_STRICT: true
    ports:
      - "8080:8080"
```

### 4.2 Starten der Container

```bash
docker compose down
docker compose up -d
docker logs -f keycloak
```

- Keycloak läuft intern auf `http://0.0.0.0:8080`
- Cloudflare Tunnel liefert extern HTTPS

## 5. Keycloak Konfiguration

1. Admin Console öffnen:

```
https://sso.pngrtz.com
```

2. Realm `demo` anlegen / auswählen

3. **Frontend URL setzen**:

```
https://sso.pngrtz.com
```

- Wichtig, damit Browser Redirects und iframe-Aufrufe HTTPS verwenden

## 6. Browser-Test

- Inkognito-Fenster nutzen
- URL: `https://sso.pngrtz.com`
- Erwartet: Keycloak Login-Seite ohne Mixed Content oder CSP Fehler
- HTTPS aktiv

## 7. Hinweise / Troubleshooting

- **Interne IP Aufrufe funktionieren nicht**, da Keycloak `KC_HOSTNAME_STRICT=true` nutzt
- Zum Testen lokal kann Hosts-Datei angepasst werden:

```
192.168.188.236   sso.pngrtz.com
```

- Cloudflare SSL Mode: **Full (strict)** empfohlen
- Logs prüfen:

```bash
docker logs keycloak --tail 50
systemctl status cloudflared
```

## 8. Aktueller Stand

- Raspberry Pi 5 läuft als Keycloak SSO-Server
- Cloudflare Tunnel liefert HTTPS unter `sso.pngrtz.com`
- Keycloak ist korrekt hinter Proxy konfiguriert
- Mixed Content / CSP Fehler behoben
