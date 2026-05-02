\## Sitzung 1: Docker + Portainer Installation



\### Schritte



\*\*Docker installieren\*\*

Docker wird über das offizielle Installationsskript eingerichtet.

```bash

curl -fsSL https://get.docker.com | sudo sh

```



\*\*Benutzer zur Docker-Gruppe hinzufügen\*\*

Damit Docker ohne `sudo` ausgeführt werden kann, wird der Benutzer `admin` 

zur Docker-Gruppe hinzugefügt. Erst nach erneutem Einloggen wird die 

Gruppenzugehörigkeit aktiv.

```bash

sudo usermod -aG docker admin

```



\*\*Installation prüfen\*\*

```bash

docker --version

\# Docker version 29.4.2, build 055a478

```



\*\*Portainer installieren\*\*

Portainer ist eine webbasierte Oberfläche zur Verwaltung von Docker-Containern.

Ein persistentes Volume wird angelegt damit Portainer-Daten einen Neustart 

überleben.

```bash

docker volume create portainer\_data

docker run -d \\

&#x20; --name portainer \\

&#x20; --restart always \\

&#x20; -p 9000:9000 \\

&#x20; -v /var/run/docker.sock:/var/run/docker.sock \\

&#x20; -v portainer\_data:/data \\

&#x20; portainer/portainer-ce:latest

```



\*\*Dashboard aufrufen\*\*

http://192.168.0.175:9000

Admin-Account angelegt. Portainer zeigt die lokale Docker-Umgebung mit 

Containern, Images und Volumes.



\### Stolpersteine

\- Docker-Gruppe erst nach erneutem SSH-Login aktiv



\### Lessons Learned

\- `--restart always` ist wichtig damit Container nach Pi-Neustart 

&#x20; automatisch wieder starten

\- `-v /var/run/docker.sock` gibt Portainer Zugriff auf Docker selbst – 

&#x20; notwendig für die Container-Verwaltung

