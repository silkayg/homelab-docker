# Homelab: Docker + Portainer + Vaultwarden auf Raspberry Pi
 
## Ziel
Raspberry Pi als Container-Host einrichten mit Docker und Portainer als
webbasierte Verwaltungsoberfläche. Vaultwarden als self-hosted Passwortmanager.
 
## Status
In Arbeit: Traefik (HTTPS) als Voraussetzung für Vaultwarden
 
## Setup
- **Hardware:** Raspberry Pi 5 (8 GB RAM)
- **OS:** Raspberry Pi OS Lite (64-bit)
- **Software:** Docker, Portainer CE, Vaultwarden
---
 
## Sitzung 1: Docker + Portainer Installation
 
### Schritte
 
**Docker installieren**
Docker wird über das offizielle Installationsskript eingerichtet.
```bash
curl -fsSL https://get.docker.com | sudo sh
```
 
**Benutzer zur Docker-Gruppe hinzufügen**
Damit Docker ohne `sudo` ausgeführt werden kann, wird der Benutzer `admin`
zur Docker-Gruppe hinzugefügt. Erst nach erneutem Einloggen wird die
Gruppenzugehörigkeit aktiv.
```bash
sudo usermod -aG docker admin
```
 
**Installation prüfen**
```bash
docker --version
# Docker version 29.4.2, build 055a478
```
 
**Portainer installieren**
Portainer ist eine webbasierte Oberfläche zur Verwaltung von Docker-Containern.
Ein persistentes Volume wird angelegt damit Portainer-Daten einen Neustart
überleben.
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
 
**Dashboard aufrufen**
```
http://192.168.0.175:9000
```
Admin-Account angelegt. Portainer zeigt die lokale Docker-Umgebung mit
Containern, Images und Volumes.
 
### Stolpersteine
- Docker-Gruppe erst nach erneutem SSH-Login aktiv
### Lessons Learned
- `--restart always` ist wichtig damit Container nach Pi-Neustart
  automatisch wieder starten
- `-v /var/run/docker.sock` gibt Portainer Zugriff auf Docker selbst –
  notwendig für die Container-Verwaltung
---
 
## Sitzung 2: Vaultwarden (abgebrochen)
 
### Schritte
 
**Vaultwarden installieren**
Self-hosted Passwortmanager, kompatibel mit Bitwarden-Clients.
```bash
docker run -d \
  --name vaultwarden \
  --restart always \
  -p 8080:80 \
  -v vaultwarden_data:/data \
  vaultwarden/server:latest
```
 
### Stolpersteine
- Vaultwarden benötigt zwingend HTTPS – auch lokal im Heimnetz
- HTTP-Workaround über `DOMAIN`-Umgebungsvariable funktioniert nicht:
  Subtle Crypto API verweigert unsicheren Kontext
### Entscheidung
Vaultwarden wird erst nach Traefik eingerichtet. Traefik stellt HTTPS
für alle lokalen Services bereit – danach kann Vaultwarden sauber
mit SSL betrieben werden.
 
### Lessons Learned
- Vaultwarden ohne HTTPS ist nicht nutzbar
- Reverse Proxy (Traefik) ist Voraussetzung für produktive Services