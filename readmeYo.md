# Instruction d'installation du serveur web

## Mettre à jour les paquets installés 

```bash
sudo dnf update
```

## Mettre SElinux en mode permissive

```bash
sudo vim /etc/selinux/config 
```

```bash
# SELINUX= can take one of these three values:
# NOTE: In earlier Fedora kernel builds, SELINUX=disabled would also
SELINUX=permissive
# SELINUXTYPE= can take one of these three values:
SELINUXTYPE=targeted
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

## Configurer la conf sshd pour écouter sur un port différent de 22


* Changer le port 

```bash
sudo vim /etc/ssh/sshd_config 
```

```bash
Port {Port_Souhaité}
```

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

sudo vim /etc/fail2ban/jail.local

sudo systemctl restart fail2ban

```

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


* Remarque 

- donner des informations sur les mots de passes 

