# raspberryPi_Webserver_Template
### Auf dem Raspberry Pi 4 (2GB) einen Webserver mit Traefik, LetsEncrypt, Wordpress und MariaDB in einer Docker-Umgebung laufen lassen. 

### Update 10.08.2020 - Neue Version mit TLS

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
https://berripi.de/raspberry-pi-webserver-mit-traefik-und-tls-in-docker/

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

    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"
    volumes:
      - "./letsencrypt:/letsencrypt"
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "./log/traefik:/log/traefik"
      - "./traefik_data/traefik.yml:/traefik.yml"
      - "./traefik_data/dynamic.yml:/dynamic.yml"
      - "/etc/localtime:/etc/localtime:ro"


  whoami:
    image: "containous/whoami"
    container_name: "whoami"
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`whoami.$DOMAIN`)"
      - "traefik.http.routers.whoami.entrypoints=websecure"
      - "traefik.http.routers.whoami.tls.certresolver=myresolver"

# Zum testen wird erst einmal der Root genutzt, das muss später unbedingt geändert werden!!!
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
      #MYSQL_USER: $MYSQL_USER 
      #MYSQL_PASSWORD: $MYSQL_PASSWORD
    ports:
      -  3306:3306 
    networks:
      -  default 
    restart:  unless-stopped 


# Zum testen wird erst einmal der Root genutzt, das muss später unbedingt geändert werden!!!
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
      #WORDPRESS_DB_USER:  root 
      WORDPRESS_DB_PASSWORD: $DBPASSWD 
      WORDPRESS_DB_NAME:  wordpress       
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.wp.rule=Host(`$DOMAIN`, `wordpress.$DOMAIN`, `www.$DOMAIN`)"
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
# .env
#Diese Variablen werden in der docker-compose.yml verwendet. Bitte alles #was zwischen den <> steht entsprechend anpassen
LOG_LEVEL=DEBUG
NETWORK_EX=web
NERWORK_INT=default

DOMAIN=<Deine Domain>

## dashboard configs
DASHBOARD_HOST="http://<Deine IP oder Domain>:8080/dashboard#/"

#Traefik
LOG=./log/traefik

#Wordpress
WPDATA=./wordpress/wp-daten
DBDATA=./wordpress/db-daten

#Hier später unbedingt einen User und ein Passwort erstellen. NICHT root!
#db config
DBUSER=root
DBPASSWD=<Dein Root Passwort>

#letsEncrypt Passwort (Das generieren wir im nächsten Schritt.
LETSENCRYPT_PW="<User:Passwort"
#Zum Beispiel so:
#<admin:$$apr1$48Hon55tF3$390fR1D0hdGZs65oBngZR6."

```
### traefik.yml erstellen
```
log:
  level: "INFO"
  filePath: "/log/traefik/traefik.log" 

providers: 
  docker: #true
    endpoint: "unix:///var/run/docker.sock"
    exposedbydefault: false
  file:
    filename: "./dynamic.yml"
    watch: true

entrypoints:
  web:
    address: :80
  websecure:
    address: :443
  dashboard:
    address: :8080

api:
  dashboard: true

certificatesresolvers:
        myresolver: 
          acme: 
          #tlschallenge: true
            httpChallenge:
              entryPoint: web
            email: "<Deine@E-Mail-Adresse>"
            storage: "/letsencrypt/acme.json"
            # Benutze Let's Encrypt Staging CA zum testen!
            # caServer: https://acme-staging-v02.api.letsencrypt.org/directory            

accessLog: {}
### dynamic.yml erstellen
```
tls:
  options:
    default:
      minVersion: VersionTLS12


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
