# Homelab: Docker + Portainer + Traefik + Vaultwarden auf Raspberry Pi 5

## Ziel
Raspberry Pi als Container-Host einrichten mit Docker und Portainer als
webbasierte Verwaltungsoberfläche. Vaultwarden als self-hosted Passwortmanager
mit echtem HTTPS über Traefik, Cloudflare DNS-Challenge und Cloudflare Tunnel
(aufgrund von CG-NAT durch Vodafone).

## Setup
- **Hardware:** Raspberry Pi 5 (8 GB RAM)
- **OS:** Raspberry Pi OS Trixie (64-bit)
- **Software:** Docker, Portainer CE, Traefik, Vaultwarden, cloudflared
- **Domain:** 4mailz.de (Strato, DNS-Verwaltung via Cloudflare)
- **Netzwerk:** Vodafone CG-NAT (kein eingehender IPv6-Traffic möglich)

---

## Architektur

```
Handy/Laptop (Heimnetz)
        ↓
Pi-hole Local DNS → 192.168.0.175 direkt

Handy/Laptop (Unterwegs)
        ↓
Cloudflare Tunnel → cloudflared → Vaultwarden
```

---

## Sitzung 1: Docker + Portainer

### Schritte

**Docker installieren**
```bash
curl -fsSL https://get.docker.com | sudo sh
```

**Benutzer zur Docker-Gruppe hinzufügen**
Erst nach erneutem Einloggen aktiv.
```bash
sudo usermod -aG docker admin
```

**Installation prüfen**
```bash
docker --version
# Docker version 29.4.2, build 055a478
```

**Portainer installieren**
Webbasierte Oberfläche zur Verwaltung von Docker-Containern.
```bash
docker volume create portainer_data

docker run -d \
  --name portainer \
  --restart always \
  -p 9000:9000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

Dashboard aufrufen:
```
http://192.168.0.175:9000
```

### Stolpersteine
- Docker-Gruppe erst nach erneutem SSH-Login aktiv

### Lessons Learned
- `--restart always` sorgt dafür dass Container nach Pi-Neustart automatisch starten
- `-v /var/run/docker.sock` gibt Portainer Zugriff auf Docker selbst

---

## Sitzung 2: Vaultwarden (erster Versuch – abgebrochen)

### Problem
Vaultwarden benötigt zwingend HTTPS, auch lokal im Heimnetz. HTTP-Workaround
über `DOMAIN`-Umgebungsvariable funktioniert nicht (Subtle Crypto API).

### Entscheidung
Vaultwarden erst nach Traefik einrichten. Traefik stellt HTTPS für alle
lokalen Services bereit.

### Lessons Learned
- Vaultwarden ohne HTTPS ist nicht nutzbar
- Reverse Proxy (Traefik) ist Voraussetzung für produktive Services

---

## Sitzung 3: Traefik + Cloudflare DNS-Challenge

### Voraussetzungen
- Domain `4mailz.de` bei Strato registriert
- DNS-Verwaltung zu Cloudflare übertragen (Nameserver bei Strato geändert)
- Cloudflare API-Token erstellt: Profile → API Tokens → Create Token → Edit Zone DNS
  - Permissions: Zone / DNS / Edit
  - Zone Resources: Include / Specific Zone / 4mailz.de

### Stolperstein: Pi-hole belegt Port 80 und 443
Pi-hole belegte standardmäßig Port 80 und 443 – Traefik konnte diese nicht nutzen.

```bash
sudo ss -tlnp | grep pihole
# LISTEN 0  200  0.0.0.0:80  ...  pihole-FTL
# LISTEN 0  200  0.0.0.0:443 ...  pihole-FTL
```

**Lösung:** Pi-hole Web-Interface auf andere Ports verschieben:
```bash
sudo nano /etc/pihole/pihole.toml
```
Ändern:
```
port = "8080o,8443os,[::]:8080o,[::]:8443os"
```
```bash
sudo systemctl restart pihole-FTL
```
Pi-hole Dashboard danach erreichbar unter `http://192.168.0.175:8080/admin`

### Traefik-Config erstellen
```bash
mkdir -p ~/traefik && nano ~/traefik/traefik.yml
```

Inhalt:
```yaml
api:
  dashboard: true
  insecure: true

entryPoints:
  web:
    address: ":80"
  websecure:
    address: ":443"

providers:
  docker:
    exposedByDefault: false

certificatesResolvers:
  cloudflare:
    acme:
      email: deine@email.de
      storage: /letsencrypt/acme.json
      dnsChallenge:
        provider: cloudflare
        resolvers:
          - "1.1.1.1:53"
          - "8.8.8.8:53"
```

### Docker-Netzwerk erstellen
```bash
docker network create traefik_network
```

### Traefik starten
```bash
docker run -d \
  --name traefik \
  --restart always \
  --network traefik_network \
  -p 80:80 \
  -p 443:443 \
  -p 8090:8080 \
  -e CF_DNS_API_TOKEN=CLOUDFLARE_TOKEN \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v ~/traefik/traefik.yml:/etc/traefik/traefik.yml \
  -v traefik_certs:/letsencrypt \
  traefik:latest
```

Dashboard: `http://192.168.0.175:8090/dashboard/`

### Lessons Learned
- Pi-hole und Traefik können nicht gleichzeitig Port 80/443 nutzen
- Pi-hole muss auf alternative Ports verschoben werden
- Cloudflare DNS-Challenge funktioniert ohne offenen Port – ideal für CG-NAT

---

## Sitzung 4: Vaultwarden mit HTTPS

### Vaultwarden starten
```bash
docker run -d \
  --name vaultwarden \
  --restart always \
  --network traefik_network \
  -v vaultwarden_data:/data \
  -e DOMAIN=https://vaultwarden.4mailz.de \
  -e SIGNUPS_ALLOWED=false \
  -l "traefik.enable=true" \
  -l "traefik.http.routers.vaultwarden.rule=Host(\`vaultwarden.4mailz.de\`)" \
  -l "traefik.http.routers.vaultwarden.entrypoints=websecure" \
  -l "traefik.http.routers.vaultwarden.tls=true" \
  -l "traefik.http.routers.vaultwarden.tls.certresolver=cloudflare" \
  -l "traefik.http.services.vaultwarden.loadbalancer.server.port=80" \
  vaultwarden/server:latest
```

### Stolperstein: CG-NAT blockiert eingehenden Traffic
`vaultwarden.4mailz.de` war von außen nicht erreichbar – Vodafone Station
blockiert eingehenden Traffic (CG-NAT).

**Lösung: Cloudflare Tunnel**

```bash
docker run -d \
  --name cloudflared \
  --restart always \
  --network traefik_network \
  --dns 1.1.1.1 \
  cloudflare/cloudflared:latest \
  tunnel --no-autoupdate run \
  --token CLOUDFLARE_TUNNEL_TOKEN
```

Cloudflare Zero Trust → Networks → Tunnels → homelab → Public Hostname:
- Subdomain: `vaultwarden`
- Domain: `4mailz.de`
- Service: `http://vaultwarden:80`

### Stolperstein: Im Heimnetz nicht erreichbar
Von außen erreichbar, aber nicht im Heimnetz (Split-DNS Problem).

**Lösung:** Lokalen DNS-Eintrag in Pi-hole setzen:
- Pi-hole Dashboard → Local DNS → DNS Records
- Domain: `vaultwarden.4mailz.de` → IP: `192.168.0.175`

Danach:
- Heimnetz → direkt über Pi
- Unterwegs → über Cloudflare Tunnel
- Überall echtes HTTPS

### Lessons Learned
- Cloudflare Tunnel löst CG-NAT ohne VPS und ohne offene Ports
- cloudflared muss im selben Docker-Netzwerk wie Vaultwarden laufen
- `SIGNUPS_ALLOWED=false` verhindert unbefugte Registrierungen
- Pi-hole Local DNS verhindert Split-DNS-Probleme im Heimnetz
