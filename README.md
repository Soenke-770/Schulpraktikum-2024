# Schulpraktikum-2024
# Installation von Nextcloud im Schulpraktikum 

  
Hallo, hier erkläre ich Schritt für Schritt, wie ich eine Cloud für mein Schulpraktikum erstellt habe.

### **Schritt 1 -** Equipment

  
Als erstes habe ich mir Hardware aus einer Kiste mit alten elektronischen Geräten ausgesucht: 

- ein    [Raspberry Pi 3er](https://www.raspberrypi.com/products/raspberry-pi-3-model-b-plus/)
- ein    [Raspberry Pi 400er](https://www.raspberrypi.com/products/raspberry-pi-400/)
- ein    [Raspberry Pi 4er](https://www.raspberrypi.com/products/raspberry-pi-4-model-b/) 

### **Schritt 2 -** Raspberry PI's installieren

  
Danach habe ich die Raspberry PI's installiert.

Der [Raspberry Pi 3er](https://www.raspberrypi.com/products/raspberry-pi-3-model-b-plus/) war defekt und konnte somit nicht verwendet werden.  
Der [Raspberry Pi 400er](https://www.raspberrypi.com/products/raspberry-pi-400/) soll als Arbeitsrechner programmiert werden.  
Auf dem [Raspberry Pi 4er](https://www.raspberrypi.com/products/raspberry-pi-4-model-b/) soll die Cloud laufen.

### **Schritt 3 -** IP Adressen und Namen

  
1\. Feste IP-Adressen zuordnen: eine für den Arbeitsrechner und eine für den Cloudrechner.  
2\. Auf dem Arbeitsrechner festlegen, dass die IP Adresse des Respberry für die Cloud jetzt "fisch" heißt. Das geht durch einen Eintrag in der /etc/hosts Datei.

### **Schritt 4 -** SSH Schlüssel erstellen

Ich habe einen SSH Schlüssel erstellt mit dem Kommando: 

```
ssh-keygen -t rsa -b 4096
```

### **Schritt 5-**SSH Schlüssel auf die Fische kopieren

Das folgende Kommando "mkdir.ssh" erstellt das Verzeichnis .ssh.

```
mkdir .ssh  
```

Das folgende Kommando "cd .ssh/" wechselt in das .ssh Verzeichnis.

```
cd .ssh/
```

Das folgende Kommando "touch authorized_keys" erzeugt eine Datei namens authorized_keys.

```
touch authorized_keys
```

Das folgende Kommando "cat ../id_rsa.pub >> authorized_keys" fügt den Inhalt der Datei id_rsa.pub an das Ende der Datei authorized_keys:

```
cat ../id_rsa.pub >> authorized_keys 
```

Das folgende Kommando "cat authorized_keys" zeigt den Inhalt der authorized_keys-Datei an.

```
cat authorized_keys
```

### **Schritt 6 -** Podman installieren

Auf der offiziellen Seite von Podman habe ich die [Anleitung](http://podman.io/docs/installation) für die Installation gefunden und die Schritte der Anleitung befolgt:  

Mit dem Kommando "apt install podman" installiert man Podman.

```
apt install podman
```

Mit dem Kommando "apt install podman-compose" installiert man podman-compose.

```
apt install podman-compose
```

Durch suchen im Internet (ich habe mich hauptsächlich nach dieser [Anleitung](https://markontech.com/posts/setup-nextcloud-with-redis-using-docker/) orientiert), Chatgpt fragen und den Hinweisen meines erfahrenen Betreuers Herrn A.  kam diese Podman-Compose-Datei zustande:

```
version: "3.9"

services:

  db-intern:
    image: docker.io/mariadb:11.3.2
    command: --transaction-isolation=READ-COMMITTED --log-bin=binlog --binlog-format=ROW
    container_name: mariadb
    restart: unless-stopped
    volumes:
      - mariadb-nextcloud-data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: "ROOT-Password"
      MYSQL_PASSWORD: "Nextcloud-Password"
      MYSQL_DATABASE: nextcloud
      MYSQL_USER: nextcloud

  redis-intern:
    image: docker.io/redis:7.2.4-alpine
    restart: unless-stopped
    container_name: redis
    command: redis-server --requirepass "das redis Password"

  nextcloud-intern:
    image: docker.io/nextcloud:28.0.4-apache
    restart: unless-stopped
    ports:
      - 80:80
    container_name: nextcloud
    environment:
      - MYSQL_HOST=db-intern
      - MYSQL_PASSWORD="Nextcloud-Password"
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - REDIS_HOST=redis-intern
      - REDIS_HOST_PASSWORD="das redis Password"
    volumes:
      - nextcloud-web:/var/www/html
      - nextcloud-config:/var/www/html/config
      - nextcloud-custom-apps:/var/www/html/custom_apps
      - nextcloud-data:/var/www/html/data
      - nextcloud-themes:/var/www/html/themes
    depends_on:
      - db-intern
      - redis-intern

volumes:
  mariadb-nextcloud-data:
  nextcloud-web:
  nextcloud-config:
  nextcloud-custom-apps:
  nextcloud-data:
  nextcloud-themes:
```

##### Erklärung der Podman-Compose-Datei:

**version**: '3.9': Gibt die Version des Compose-Formats an.

**services**: Definiert die drei Services nextcloud, db, und redis.

**db-intern**

- **image**: Verwendet das neueste MariaDB-Image.
- **command**: Das Kommando wird nach dem erzeugen im Container 1mal ausgeführt 
- **restart**: startet den Container neu wenn er abstürzt 
- **container_name**: Name des Containers.
- **environment**: Setzt Umgebungsvariablen für die Konfiguration von MariaDB.
  - **MYSQL_ROOT_PASSWORD**: ROOT-Passwort für MariaDB.
  - **MYSQL_DATABASE**: Name der Datenbank.
  - **MYSQL_USER**: Datenbankbenutzername.
  - **MYSQL_PASSWORD**: Datenbankpasswort.

**redis-intern:**

- **image**: Verwendet das `redis:alpine` Image.
- **restart**: startet den Container neu wenn er abstürzt
- **container_name**: Name des Containers.
- **command**: Startet Redis im append-only mode.

**nextcloud**:

- **image**: Verwendet das neueste Nextcloud-Image.
- **restart**: startet den Container neu wenn er abstürzt
- **ports**: Mappt den Port 8080 des Hosts auf den Port 80 des Containers.
- **container_name**: Name des Containers.
- **environment**: Setzt Umgebungsvariablen für die Konfiguration von Nextcloud.
  - **MYSQL_HOST**: Hostname des MariaDB-Containers.
  - **MYSQL_DATABASE**: Name der Datenbank.
  - **MYSQL_USER**: Datenbankbenutzername.
  - **MYSQL_PASSWORD**: Datenbankpasswort.
  - **REDIS_HOST**: Hostname des Redis-Containers.
- **volumes**: Bindet das benannte Volume `nextcloud_data` an das Verzeichnis `/var/www/html` im Container.
- **depends_on**: Stellt sicher, dass die Container `db` und `redis` vor dem Start von Nextcloud gestartet werden.

**volumes**: Definiert benannte Volumes `nextcloud_data`, `db_data`, und `redis_data`, die von den Containern verwendet werden.

### **Schritt 7: Installation von Nextcloud**

  
Ich habe über das Netzwerk Zugriff auf "fisch" verschafft und habe dann mit folgendem Kommando Nextcloud über Podman installiert und gestartet.

```
podman-compose up -d
```


