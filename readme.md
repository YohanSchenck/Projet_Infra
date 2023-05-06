
# Table des matières

1. [Prérequis](#prérequis)
2. [Intructions d'installation du serveur proxy](#instructions-dinstallation-du-serveur-proxy)
    1. [Configuration SSH](#configuration-ssh)
    2. [Installation de nginx/parametrage du reverse proxy](#installation-de-nginxparametrage-du-reverse-proxy)
    3. [Installation et Configuration Fail2Ban](#installation-et-configuration-fail2ban)
    4. [Installation et configuration du WAF](#installation-et-configuration-du-waf)
    5. [Installer et configurer Netdata](#installer-et-configurer-netdata)
3. [Intructions d'installation du serveur web](#instructions-dinstallation-du-serveur-web)
    1. [Configurer le firewalld](#configurer-le-firewalld)
    2. [Configurer la conf sshd](#configurer-la-conf-sshd)
    3. [Installer et configurer Fail2Ban](#installer-et-configurer-fail2ban)
    4. [Installation de Jirafeau](#installation-de-jirafeau)
    5. [Installer Netdata](#installer-netdata)
    6. [Installer Grafana](#installer-grafana)
    7. [Installer InfluxDB](#installer-influxdb)
    8. [Configurer netdata afin d'utiliser InfluxDB](#configurer-netdata-afin-dutiliser-influxdb)
    9. [Configurer Grafana afin d'utiliser InfluxDB](#configurer-grafana-afin-dutiliser-influxdb)
4. [Instructions d'accès](#instruction-daccès)


# Schéma de l'architecture

![Schéma_Archi](/Sch%C3%A9ma_Archi.png)

# Prérequis

* Avoir 2 VPS distincts
    * Accessible via SSH
    * hostname défini
    * SELinux en mode "permissive"


# Instructions d'installation du serveur proxy

**Les commandes suivantes sont à réaliser sur le serveur proxy**

## Configuration SSH

```bash
sudo vim /etc/ssh/sshd_config
```
Dans le fichier **sshd_config** modifier :

- La connetion root
```
PermitRootLogin no
```
- Le Port et le protocol
```
Port <Celui que vous voulez mais pas 22>
Protocol 2
```
- Le nombre max des session connecter et le nombre de tentative de connection
```
MaxAuthTries <le nombre de vous voulez <5 >
MaxSessions <le nombre de vous voulez <5 >
```
- Les personnes authoriser a ce connecter
```
AllowUsers <personne1> <personne2>
```
- La connection par cle
```
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
```
- La connection par mot de passe et mot de passe vide
```
PasswordAuthentification no
PermitEmptyPasswords no
```
- Le temps max d'inactiviter et le nombre de relance avant kick pour inactiviter
```
ClientAliveInterval <temps en seconde>
ClientAliveCountMax <nombre de relance>
```
- Une baniere 
```
Banner <chemin du fichier avec votre banner>
```

## Installation de nginx/parametrage du reverse proxy

```
sudo apt update
sudo apt install nginx
sudo systemctl start nginx
sudo systemctl enable nginx

sudo apt install certbot
sudo apt install certbot python3-certbot-nginx
certbot certonly --rsa-key-size 4096 --nginx


sudo nano /etc/nginx/sites-enabled/default 
server {
    listen 80;
    server_name farkastheduck.online www.farkastheduck.online;
    return 301 https://www.farkastheduck.online;
}

server {
    listen 443 ssl;
    server_name farkastheduck.online www.farkastheduck.online;
    modsecurity on;
    modsecurity_rules_file /etc/nginx/modsec/main.conf;
    ssl_certificate /etc/letsencrypt/live/farkastheduck.online/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/farkastheduck.online/privkey.pem;

    location / {
        proxy_pass <IP_SERVEUR_WEB>;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection keep-alive;
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

sudo systemctl restart nginx

```

## Installation et Configuration Fail2Ban

**Installation**

```powershell
sudo dnf install epel-release

sudo dnf install fail2ban fail2ban-firewalld

sudo systemctl enable fail2ban

sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local

sudo systemctl start fail2ban
```


**Configuration**

```bash
sudo vim /etc/fail2ban/jail.local
```
Dans le fichier **jail.local** modifier :

- le temps de bannissement à infini 
```
bantime = -1
```
- le nombre de tentative 
```
maxretry = 3
```
- le temps pour la detection de ban ( dans ce cas 3 essais en moins de 10 min)
```
findtime = 10m
```
- la conf sshd
```
[sshd]

enabled = true 
port = ssh
backend = %(sshd_backend)s
logpath = %(sshd_log)s
maxretry = 3
```

## Installation et configuration du WAF

* Prérequis 

installer git 
```bash
sudo apt install git
```
installer tous pour build et compiler 

```bash
sudo apt-get install bison build-essential ca-certificates curl dh-autoreconf doxygen \
  flex gawk git iputils-ping libcurl4-gnutls-dev libexpat1-dev libgeoip-dev liblmdb-dev \
  libpcre3-dev libpcre++-dev libssl-dev libtool libxml2 libxml2-dev libyajl-dev locales \
  lua5.3-dev pkg-config wget zlib1g-dev zlibc libxslt libgd-dev
```


* Installer ModSecurity 

cloner le repo git du projet
```bash
cd /opt && sudo git clone https://github.com/SpiderLabs/ModSecurity 
```
build ModSecurity
```bash
cd ModSecurity && sudo git submodule init && sudo git submodule update
sudo ./build.sh
sudo ./configure
sudo make
sudo make install
```

* Installer le connecteur ModSecurity-Nginx

```bash
cd /opt && sudo git clone --depth 1 https://github.com/SpiderLabs/ModSecurity-nginx.git
```

* Installer le module ModSecurity sur Nginx

Recuperer la version de Nginx

```bash
nginx -v
```
Exemple de output : nginx version: nginx/**1.14.0**

```bash
cd /opt && sudo wget http://nginx.org/download/nginx-<version de nginx>.tar.gz
sudo tar -xvzmf nginx-<version de nginx>.tar.gz
 cd nginx-<version de nginx>
```
Recuperer les arguments de configuration de Nginx
```bash
nginx -V
```
Exemple de output : nginx version: nginx/1.14.0 (Ubuntu)
built with OpenSSL 1.1.1  11 Sep 2018
TLS SNI support enabled
configure arguments: **--with-cc-opt='-g -O2 -fdebug-prefix-map=/build/nginx-GkiujU/nginx-1.14.0=. [...] --with-mail=dynamic --with-mail_ssl_module**
```bash
sudo ./configure --add-dynamic-module=../ModSecurity-nginx <Configure Arguments>
sudo make modules
sudo mkdir /etc/nginx/modules
sudo cp objs/ngx_http_modsecurity_module.so /etc/nginx/modules
```

* Configurer le Module ModSecurity
```bash
sudo vim /etc/nginx/nginx.conf
```
Ajouter cette ligne 
**load_module /etc/nginx/modules/ngx_http_modsecurity_module.so;**
pour obtenir ceci
```
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;
load_module /etc/nginx/modules/ngx_http_modsecurity_module.so;
```
* Configurer le OWASP-CRS
```bash
sudo rm -rf /usr/share/modsecurity-crs
sudo git clone https://github.com/coreruleset/coreruleset /usr/local/modsecurity-crs
sudo mv /usr/local/modsecurity-crs/crs-setup.conf.example /usr/local/modsecurity-crs/crs-setup.conf
sudo mv /usr/local/modsecurity-crs/rules/REQUEST-900-EXCLUSION-RULES-BEFORE
```

* Configurer ModSecurity

```bash
sudo mkdir -p /etc/nginx/modsec
sudo cp /opt/ModSecurity/unicode.mapping /etc/nginx/modsec
sudo cp /opt/ModSecurity/modsecurity.conf-recommended /etc/nginx/modsec/modsecurity.conf
```
Changer la valeur de **SecRuleEngine** en **On** dans le fichier **modsecurity.conf**
```bash
sudo vim /etc/nginx/modsec/modsecurity.conf
```
Ajouter **Include /etc/nginx/modsec/modsecurity.conf
Include /usr/local/modsecurity-crs/crs-setup.conf
Include /usr/local/modsecurity-crs/rules/*.conf** dans le fichier **main.conf**
```bash
sudo touch /etc/nginx/modsec/main.conf
sudo vim /etc/nginx/modsec/main.conf
```

* Configurer Nginx

```bash
sudo vim /etc/nginx/sites-available/default
```

Dans le fichier **default** ajouter **modsecurity on;
modsecurity_rules_file /etc/nginx/modsec/main.conf;** pour ressembler a ceci 

```
server {
        listen 80 default_server;
        listen [::]:80 default_server;

        root /var/www/html;

        modsecurity on;
        modsecurity_rules_file /etc/nginx/modsec/main.conf;

        index index.html index.htm index.nginx-debian.html;

        server_name _;
        location / {
                try_files $uri $uri/ =404;
        }
}
```
Redemarrer Nginx 
```bash 
sudo systemctl restart nginx
```

Votre WAF ModSecurity est installer sur votre Nginx


* Informations sur le WAF

**Fichier de log ModSecurity**

```bash 
sudo vim /var/log/modsec_audit.log
```

**Dossier des regles de ModSecurity**

```bash
sudo vim /usr/local/modsecurity-crs/rules/
```


## Installer et configurer Netdata

**Installation**

* Installer le repository EPEL

```bash
sudo dnf install epel-release -y
```

* Installer les packages nécessaire pour netdata

```bash
sudo wget -O /tmp/netdata-kickstart.sh https://my-netdata.io/kickstart.sh && sh /tmp/netdata-kickstart.sh
```

* Démarrer le service netdata

```bash
sudo systemctl start netdata

sudo systemctl enable netdata
```

* Ouvrir le port 19999

```bash
sudo firewall-cmd --add-port=19999/tcp --permanent

sudo firewall-cmd --reload
```

**Configuration**

Configurer Netdata afin d'autoriser l'envoi des metrics vers le serveur web

* Autoriser l'exporting engine

Dans le dossier /etc/netdata

```bash
sudo ./edit-config exporting.conf
```

Modifier les lignes suivantes du fichier exporting.conf

```bash
[exporting:global]
    enabled = yes
```

* Autoriser l'envoi de données vers une base de donnée OpenTSB

Ajouter les lignes suivantes au fichier exporting.conf

```bash
[opentsdb:opentsdb]
    enabled = yes
    destination = <IP_SERVEUR_WEB>:4242
    prefix = proxy
```

* Redémarrer le service netdata

```bash
sudo sysytemctl restart netdata
```


# Instructions d'installation du serveur web

**Les commandes suivantes sont à réaliser sur le serveur web**

## Mettre à jour les paquets installés 

```bash
sudo dnf update
```

## Configurer le firewalld

Restreindre les ports d'écoute

```bash
sudo firewall-cmd --set-default-zone=drop
```

Ajouter des règles pour ssh 

```bash
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="0.0.0.0/0" port port="<Port_Souhaité>" protocol="tcp" accept'
```

## Configurer la conf sshd 


**Se référer à la configuration ssh du serveur proxy**

* Relancer le serivce ssh

```bash
sudo systemctl restart ssh.service
```


## Installer et configurer Fail2Ban

```powershell
sudo dnf install epel-release

sudo dnf install fail2ban fail2ban-firewalld

sudo systemctl enable fail2ban

sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local

sudo systemctl start fail2ban

```

**Pour la configuration se référer à la configuration Fail2ban du serveur proxy**

## Installation de Jirafeau

* Installer php et git

```bash
sudo dnf install php

sudo dnf install git 

```

* Cloner le repo dans /var/www

```bash

git clone https://gitlab.com/mojo42/Jirafeau.git

cp /Jirafeau/lib/config.original.php /Jirafeau/lib/config.local.php
```

* Modifier le fichier de config si besoin

```bash
sudo vim /var/www/Jirafeau/lib/config.local.php 
```

* Donner la propriété et les permissions à l'utilisateur démarrant le serveur web (généralement apache)

```bash
sudo chown -R apache /var/www/Jirafeau
```

* Installation et configuration du serveur web

- Installation

```bash
sudo dnf install nginx
```

- Créer un dossier ssl

```bash
sudo mkdir /etc/nginx/ssl
cd /etc/nginx/ssl/
```

- Générer les certificats

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /path/to/key.key -out /path/to/certificate.crt
```

- Configuration

```bash
sudo vim /etc/nginx/nginx.conf
```
[nginx.conf](./nginx_Web.conf)


* Configurer le firewall

```bash
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="<IP_SERVEUR_PROXY>" port port="443" protocol="tcp" accept'

sudo firewall-cmd --permanent --reload
```


* Démarrer le service nginx
```bash
sudo systemctl start nginx

sudo systemctl enable nginx
```

**Informations sur Jirafeau**

- Les fichiers sont sauvegardés dans /var/www/Jirafeau/var/files

- Les liens et le mot de passe hashé sont sauvegardés dans /var/www/Jirafeau/var/links


## Installer Netdata

* Installer le repository EPEL

```bash
sudo dnf install epel-release -y
```

* Installer les packages nécessaire pour netdata

```bash
sudo wget -O /tmp/netdata-kickstart.sh https://my-netdata.io/kickstart.sh && sh /tmp/netdata-kickstart.sh
```

* Démarrer le service netdata

```bash
sudo systemctl start netdata

sudo systemctl enable netdata
```

* Ouvrir le port 19999

```bash
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="0.0.0.0/0" port port="19999" protocol="tcp" accept'
firewall-cmd --reload
```

## Installer Grafana

* Ajouter le repository Grafana

```bash
sudo vim /etc/yum.repos.d/grafana.repo
```

[grafana.repo](./grafana.repo)

* Vérifier le package de grafana dans le repository officiel

```bash
sudo info grafana
```

* Installer grafana

```bash
sudo dnf install grafana -y
```

* Démarrer le service grafana

```bash
sudo systemctl start grafana-server

sudo systemctl enable grafana-server
```

* Ouvrir le port 3000

```bash
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="0.0.0.0/0" port port="3000" protocol="tcp" accept'
sudo firewall-cmd --reload
```

## Installer influxdb


* Ajouter le repository d'Influxdb

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/influxdb.repo
[influxdb]
name = InfluxDB Repository - RHEL \$releasever
baseurl = https://repos.influxdata.com/rhel/\$releasever/\$basearch/stable
enabled = 1
gpgcheck = 1
gpgkey = https://repos.influxdata.com/influxdb.key
EOF
```

* Installer Influxdb

```bash
sudo dnf -y install influxdb
```

* Démarrer Influxdb

```bash
sudo systemctl start influxdb

sudo systemctl enable influxdb
```

* Ajouter un utilisateur admin

Démarrer le client influxdb

```bash
influx
```


```bash
CREATE USER admin WITH PASSWORD 'Votre_Mdp_Personnel' WITH ALL PRIVILEGES
```

* Ouvrir le port 4242 pour recevoir les metrics netdata du serveur proxy

```bash
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="<IP_SERVEUR_PROXY>" port port="4242" protocol="tcp" accept'
firewall-cmd --reload
```


## Configurer netdata afin d'utiliser InfluxDB

* Autoriser l'exporting engine

Dans le dossier /etc/netdata

```bash
sudo ./edit-config exporting.conf
```

Modifier les lignes suivantes du fichier exporting.conf

```bash
[exporting:global]
    enabled = yes
```

* Autoriser l'envoi de données vers une base de donnée OpenTSB

Ajouter les lignes suivantes au fichier exporting.conf

```bash
[opentsdb:opentsdb]
    enabled = yes
    destination = localhost:4242
    prefix = web
```

* Autoriser le service OpenTSDB dans InfluxDB

```bash
sudo vim /etc/influxdb/influxdb.conf
```

Ajouter/Modifier ces lignes

```bash
[[opentsdb]]
   enabled = true
   bind-address = ":4242"
   database = "opentsdb"
```

* Rédémarrer Netdata et InfluxDB

```bash
sudo systemctl restart influxdb netdata 
```

Netdata doit normalement envoyer ses metrics à InfluxDB.

## Configurer Grafana afin d'utiliser InfluxDB

* Se Connecter à Grafana avec l'ip suivante :

- http://Ip_Du_Serveur_Web:3000


* Créer un compte admin  et se connecter à ce compte.

* Ajouter une source de données InfluxDB. 

Indiquer :
- L'URL = http://localhost:8086
- La base de données = opentsdb
- User = admin
- Password = 'Votre_Mdp_Personnel'

* Ajouter des metrics dans un dashboard


# Instruction d'accès 

* Vous pouvez accéder à la solution via l'URL suivant

**https://<IP_SERVEUR_PROXY OU NOM_DE_DOMAINE_SERVEUR_PROXY>**
