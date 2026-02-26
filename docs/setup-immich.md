# Installation von Immich auf Raspberry Pi 5 mit Cloudflare Tunnel + Keycloak

## 1. Ziel

Eine Immich Instanz auf einem Raspberry Pi 5, erreichbar über eine externe Domain (`immich.pngrtz.com`), mit HTTPS über Cloudflare Tunnel.

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

1. **Tunnel erstellen**

```bash
cloudflared tunnel create immich
```

- Tunnel-ID wird angezeigt: `---`

1. **Tunnel-DNS-Route erstellen**

```bash
cloudflared tunnel route dns immich immich.pngrtz.com
```

- DNS-Eintrag (CNAME) wird bei Cloudflare gesetzt

1. **Tunnel-Konfig speichern**

- Datei: `/etc/cloudflared/config.yml`

```yaml
tunnel: immich
credentials-file: /root/.cloudflared/---.json

ingress:
  - hostname: immich.pngrtz.com
    service: http://localhost:2283
  - service: http_status:404
```

1. **Tunnel als Service starten**

```bash
sudo cloudflared service install
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
systemctl status cloudflared
```

- Status zeigt `active (running)`

## 4. Docker / Immich Setup

### 4.1 Docker Compose und .env File

Pfad: `~/immich-app/docker-compose.yml`

```yaml
name: immich

services:
  immich-server:
    container_name: immich_server
    image: ghcr.io/immich-app/immich-server:${IMMICH_VERSION:-release}
    # extends:
    #   file: hwaccel.transcoding.yml
    #   service: cpu # set to one of [nvenc, quicksync, rkmpp, vaapi, vaapi-wsl] for accelerated transcoding
    volumes:
      # Do not edit the next line. If you want to change the media storage location on your system, edit the value of UPLOAD_LOCATION in the .env file
      - ${UPLOAD_LOCATION}:/data
      - /etc/localtime:/etc/localtime:ro
    env_file:
      - .env
    ports:
      - '2283:2283'
    depends_on:
      - redis
      - database
    restart: always
    healthcheck:
      disable: false

  immich-machine-learning:
    container_name: immich_machine_learning
    # For hardware acceleration, add one of -[armnn, cuda, rocm, openvino, rknn] to the image tag.
    # Example tag: ${IMMICH_VERSION:-release}-cuda
    image: ghcr.io/immich-app/immich-machine-learning:${IMMICH_VERSION:-release}
    # extends: # uncomment this section for hardware acceleration - see https://docs.immich.app/features/ml-hardware-acceleration
    #   file: hwaccel.ml.yml
    #   service: cpu # set to one of [armnn, cuda, rocm, openvino, openvino-wsl, rknn] for accelerated inference - use the `-wsl` version for WSL2 where applicable
    volumes:
      - model-cache:/cache
    env_file:
      - .env
    restart: always
    healthcheck:
      disable: false

  redis:
    container_name: immich_redis
    image: docker.io/valkey/valkey:9@sha256:546304417feac0874c3dd576e0952c6bb8f06bb4093ea0c9ca303c73cf458f63
    healthcheck:
      test: redis-cli ping || exit 1
    restart: always

  database:
    container_name: immich_postgres
    image: ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0@sha256:bcf63357191b76a916ae5eb93464d65c07511da41e3bf7a8416db519b40b1c23
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_USER: ${DB_USERNAME}
      POSTGRES_DB: ${DB_DATABASE_NAME}
      POSTGRES_INITDB_ARGS: '--data-checksums'
      # Uncomment the DB_STORAGE_TYPE: 'HDD' var if your database isn't stored on SSDs
      # DB_STORAGE_TYPE: 'HDD'
    volumes:
      # Do not edit the next line. If you want to change the database storage location on your system, edit the value of DB_DATA_LOCATION in the .env file
      - ${DB_DATA_LOCATION}:/var/lib/postgresql/data
    shm_size: 128mb
    restart: always
    healthcheck:
      disable: false

volumes:
  model-cache:
```

Anschließend müssen die Passwörter vergeben werden. Dies geschieht in der **.env-Datei**

```yaml
# You can find documentation for all the supported env variables at https://docs.immich.app/install/environment-variables

# The location where your uploaded files are stored
UPLOAD_LOCATION=./library

# The location where your database files are stored. Network shares are not supported for the database
DB_DATA_LOCATION=./postgres

# To set a timezone, uncomment the next line and change Etc/UTC to a TZ identifier from this list: https://en.wikipedia.org/wiki/List_of_tz_database_time_zones#List
# TZ=Etc/UTC

# The Immich version to use. You can pin this to a specific version like "v2.1.0"
IMMICH_VERSION=v2

# Connection secret for postgres. You should change it to a random password
# Please use only the characters `A-Za-z0-9`, without special characters or spaces
DB_PASSWORD=postgres

# The values below this line do not need to be changed
###################################################################################
DB_USERNAME=postgres
DB_DATABASE_NAME=immich
```
Die *docker-compose.yml* setzt während des startens die jeweiligen Werte der .env Datei in die geschweiften Klammern ein. 

## 4.2 Starten der Container

```bash
docker compose down
docker compose up -d
```
- Immich läuft nun intern auf `0.0.0.0:2283``
- Cloudflare Tunnel liefert extern HTTPS

## 4.3 Einrichten von Immich

1. Immich öffnen
``` URL
immich.pngrtz.com
```
Wenn kein Admin-Account vorhanden ist, fordert immich den Nutzer dazu auf einen Anzulegen. Hier ist eine E-Mail Adresse und ein Passwort dafür zu vergeben. 

2. OAuth in Immich aktivieren und konfigurieren

Im nächsten Schritt muss in Immich intern die OAuth funktion aktiviert werden. Hierfür muss man mit einem Admin-Account eingeloggt sein. 
Hierfür:
Profilbild -> Verwaltung -> Authentifizierungseinstellungen -> OAuth -> Anmeldung mit OAuth

Von dieser Stelle aus fügt man die *issuer_url* und die *client_id* aus Keycloak ein.

## 5. Browser-Test

- Inkognito-Fenster nutzen
- URL: `https://immich.pngrtz.com`
- Erwartet: Immich Login-Seite mit OAuth Login Option
- HTTPS aktiv

## 6. Aktueller Stand

- muss noch alles ausgefürt werden