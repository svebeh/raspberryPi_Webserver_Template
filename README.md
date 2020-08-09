# raspberryPi_Webserver_Template
### Auf dem Raspberry Pi 4 (2GB) einen Webserver mit Traefik, LetsEncrypt, Wordpress und MariaDB in einer Docker-Umgebung laufen lassen. 

Seit vielen Jahren hatte ich einen V-Server bei einem Internet Provider gemietet. Der ganze Spaß hatte ca. 10 Euro/mtl. gekostet. Den Server habe ich nun gekündigt.

Jetzt wollte ich wissen, ob es möglich ist, mit wenig Aufwand und zu geringen Kosten einen eigenen Webserver, der aus dem Internet erreichbar ist, zu Hause aufzubauen.

Um es vorweg zu nehmen… Ja, es hat funktioniert! Was habe ich dafür benötigt?

### WAS WIRD BENÖTIGT
Raspberry Pi 4 (2GB)
Eine vernünftige Micro SD-Karte (z.B. ScanDisk 16, 32 oder 64 GB.)
Eine Domain (bei einem Provider der DynDns unterstützt. Ich bin bei Strato)

Auf dem Raspberry Pi sollte natürlich schon ein Raspberry Pi OS laufen!

Wie man Docker auf dem Raspberry Pi installiert, kannst Du auf meiner Webserite sehen https://berripi.de/docker-auf-raspberry-pi/
Hier die kurze Abfolge der Befehle:
```
# Updates installieren
sudo apt-get update 
sudo apt-get upgrade 

# System Neustarten
sudo reboot

# Abhängigkeiten installieren (falls noch nicht vorhanden)
sudo apt-get install wget git apt-transport-https vim telnet 

# Docker Installer downloaden 
curl -sSL https://get.docker.com | sh

# Docker installieren
sudo sh get-docker.sh

# User (z.B. pi) zur Gruppe Docker hinzufügen 
sudo usermod -aG docker ${USER} 

# Docker aktivieren und starten 
sudo systemctl enable docker 
sudo systemctl start docker 

# Docker Version ausgeben 
docker version 
```

Eine genauere Anleitung ist auf meiner Webseite zu finden. Dort beschreibe ich auf wie man den Router konfigurieren muss und welche DNS-Einstellungen beim Provider vorgenommen werden müssen.
https://berripi.de/docker-portainer-traefik-wordpress-mariadb-auf-raspberry-pi/

Hier gibt es jetzt erst einmal nur die Abfolge der Befehle und wie mal die docker-compose.yml erstellt.

### Docker-Compose installieren
```
# Abhängigkeiten installieren
sudo apt-get install libffi-dev libssl-dev
sudo apt install python3-dev
sudo apt-get install -y python3 python3-pip

# Docker-Compose installieren
sudo pip3 install docker-compose

# Docker-Compose ausführbar machen
sudo chmod +x /usr/local/bin/docker-compose
```

### docker-compose.yml erstellen (An den Stellen mit Kommentaren müssen Anpassungen vorgenommen werden!)
```
version: "3.3"

services:

  traefik:
    image: "traefik:v2.2"
    container_name: "traefik"
    command:
      - "--api.insecure=true" #<-- Das sollte in PROD deaktiviert werden weil das Dashboard sonst von außen erreichbar ist!!!
      - "--api.dashboard=true" 
      - "--log.level=INFO"
      - "--accesslog=true"
      - "--log.filePath=/log/traefik/traefik.log" 
      - "--log.filePath=/log/traefik/access.log"
        #- "--log.format=json" 
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.myresolver.acme.tlschallenge=true"
      # Fuer Development kann man ungueltige Zertifikate über den Link nutzen
      #- "--certificatesresolvers.myresolver.acme.caserver=https://acme-staging-v02.api.letsencrypt.org/directory"
      - "--certificatesresolvers.myresolver.acme.email=webmaster@$DOMAIN" #<-- den forderen Teil der E-Mail ersetzen
      - "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
    ports:
      - "443:443"
      - "8080:8080"
    volumes:
      - "./letsencrypt:/letsencrypt"
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "./log/traefik:/log/traefik"
   


  whoami:
    image: "containous/whoami"
    container_name: "whoami"
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`whoami.$DOMAIN`)" #<-- ggf. die Subdomain anpassen
      - "traefik.http.routers.whoami.entrypoints=websecure"
      - "traefik.http.routers.whoami.tls.certresolver=myresolver"


  db:
    depends_on: 
      -  traefik 
    image:  linuxserver/mariadb:arm32v7-latest 
    container_name:  wp_mariadb 
    volumes:
      -  $DBDATA:/config 
    environment:
      TZ:  Europe/Berlin 
      MYSQL_ROOT_PASSWORD: $DBPASSWD 
      MYSQL_DATABASE:  wordpress 
    ports:
      -  3306:3306 
    networks:
      -  default 
    restart:  unless-stopped 



  wp:
    depends_on: 
      -  db 
    links: 
      -  db 
      -  db:wp_mariadb   
    image:  wordpress:latest 
    container_name:  wordpress 
    volumes:
      -  $WPDATA:/var/www/html 
    environment:
      WORDPRESS_DB_HOST:  db:3306 
      #WORDPRESS_DB_USER:  root  #<-- zum Testen mit root
      WORDPRESS_DB_PASSWORD: $DBPASSWD 
      WORDPRESS_DB_NAME:  wordpress   
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.wp.rule=Host(`$DOMAIN`, `wordpress.$DOMAIN`)" #<-- ggf. die Subdomain anpassen
      - "traefik.http.routers.wp.entrypoints=websecure"
      - "traefik.http.routers.wp.tls.certresolver=myresolver"
    networks:
      -  default 
      -  web 
    restart:  unless-stopped 


networks:
  web:
    external: true

```

### Verschlüsseltes Passwort für das Dashboard erstellen
Das Passwort muss dann in der .env eingefügt werden!
```
#Apache Tools downloaden 
sudo apt-get install apache2-utils

#Passwort erstellen
echo $(htpasswd -nbB <DeinUsername> "<DeinPasswort>") | sed -e s/\\$/\\$\\$/g

```
### .env erstellen. (Hier müssen einige Variablen angepasst werden!)
```
LOG_LEVEL=ERROR
NETWORK_EX=web
NERWORK_INT=default

DOMAIN=<DEINE_DOMAIN>    #Hier anpassen!

##Dashboard config
DASHBOARD_HOST="http://<DEINE_IP_ADRESSE>:8080/dashboard#/"    #Hier anpassen!

#Traefik
LOG=./log/traefik

#Wordpress
WPDATA=./wordpress/wp-daten
DBDATA=./wordpress/db-daten

#db config
DBUSER=<DEIN_DB_USER>    #Hier anpassen!
DBPASSWD=<DEIN_DB_PASSWORT>    #Hier anpassen!

#Passwort für Dashboard
LETSENCRYPT_PW="<DEIN_USER:wefprfew$$ewfon534tgrrgj1$394tgrefRrg5cpyGQ54greroBngIX1."    #Hier anpassen!

```

Wenn das alles erledigt ist, dann wird die Umgebung hochgefahren mit
```
#Umgebung starten
docker-compose up -d

#Umgebung stoppen
docker-compose down
```

### Portainer
Es gibt noch ein schönes Webinterface (Portainer) über das man sämtliche Container verwalten kann. Porttainer kann man so starten:

```
sudo docker run -d -p 9000:9000 --restart always --name portainer -v portainer_data:/data -v /var/run/docker.sock:/var/run/docker.sock portainer/portaine

```
Portainer lässt sich dann im Browser über http://<DEINE-IP-ADRESSE:9000 starten.

Good luck und Ahoi! :-)
