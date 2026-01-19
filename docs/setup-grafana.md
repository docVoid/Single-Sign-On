# Installation von Grafana auf Raspberry Pi 5 mit Cloudflare Tunnel + Keycloak

## 1. Ziel

EineGrafana Instanz auf einem Raspberry Pi 5, erreichbar über eine externe Domain (`monitor.pngrtz.com`), mit HTTPS über Cloudflare Tunnel.

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
cloudflared tunnel create grafana
```

- Tunnel-ID wird angezeigt: `---`

3. **Tunnel-DNS-Route erstellen**

```bash
cloudflared tunnel route dns grafana monitor.pngrtz.com
```

- DNS-Eintrag (CNAME) wird bei Cloudflare gesetzt

4. **Tunnel-Konfig speichern**

- Datei: `/etc/cloudflared/config.yml`

```yaml
tunnel: grafana
credentials-file: /root/.cloudflared/---.json

ingress:
  - hostname: monitor.pngrtz.com
    service: http://localhost:3000
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

## 4. Docker / Grafana Setup

### 4.1 Docker Compose

Pfad: `~/grafana/docker-compose.yml`

```yaml
version: "3.8"

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    networks:
      - ???? ##needs to be checked

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    environment:
      - GF_SERVER_ROOT_URL=https://monitor.pngrtz.com
      - GF_SERVER_HTTP_PORT=3000
      - GF_AUTH_GENERIC_OAUTH_ENABLED=true
      - GF_AUTH_GENERIC_OAUTH_NAME=Keycloak
      - GF_AUTH_GENERIC_OAUTH_ALLOW_SIGN_UP=true
      - GF_AUTH_GENERIC_OAUTH_CLIENT_ID=grafana
      - GF_AUTH_GENERIC_OAUTH_SCOPES=openid profile email
      - GF_AUTH_GENERIC_OAUTH_CLIENT_SECRET=CLIENT_ID_SECRET_HERE
      - GF_AUTH_GENERIC_OAUTH_AUTH_URL=https://sso.pngrtz.com/realms/RBS-Ulm/protocol/openid-connect/auth
      - GF_AUTH_GENERIC_OAUTH_TOKEN_URL=http://keycloak:8080/realms/RBS-Ulm/protocol/openid-connect/token
      - GF_AUTH_GENERIC_OAUTH_API_URL=http://keycloak:8080/realms/RBS-Ulm/protocol/openid-connect/userinfo
      - GF_AUTH_GENERIC_OAUTH_TLS_SKIP_VERIFY_INSECURE=true
      - GF_AUTH_GENERIC_OAUTH_ROLE_ATTRIBUTE_PATH=contains(resource_access.grafana.roles[*], 'admin') && 'Admin' || 'Viewer'
    networks:
      - default
      - keycloak_net
    ports:
      - "3000:3000"
    restart: always

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    ports:
      - "9100:9100"
    restart: always

networks:
  keycloak_net:
    external: true
    name: keycloak_default

```

### 4.2 Starten der Container

```bash
docker compose down
docker compose up -d
docker logs -f grafana
```

- Grafana läuft intern auf `http://0.0.0.0:3000`
- Cloudflare Tunnel liefert extern HTTPS

## 5. Grafana Konfiguration

1. Grafana öffnen:

```
https://monitor.pngrtz.com
```
2. Datenquelle hinzufügen:

2.1. Auf Verbindungen -> Neu Verbindung hinzufügen -> Prometheus

2.2. Prometheus Server URL setzten:

```
http://prometheus:9090
```

2.3. Auf Speichern und Testen drücken

3. Dashboard hinzufügen

3.1. Import von Grafana.com
3.2. ID: 1860 eingeben
3.3. Aud Laden clicken und Prometheus als Datenquelle auswählen

## 6. Browser-Test

- Inkognito-Fenster nutzen
- URL: `https://monitor.pngrtz.com`
- Erwartet: Grafana Login-Seite
- HTTPS aktiv

## 7. Hinweise / Troubleshooting

- Keycoak client configuratiokn findet zum größten Teil in der docker-compose.yml statt.

## 8. Aktueller Stand

- Cloudflare Tunnel liefert HTTPS unter `monitor.pngrtz.com`
- Grafana ist korrekt hinter Proxy konfiguriert
