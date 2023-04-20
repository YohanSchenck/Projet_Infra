# Instruction d'installation du serveur web

* Mettre à jour les paquets installés 

```bash
sudo dnf update
```

* Mettre SElinux en mode permissive

```bash
[rocky@vps-d0e86d3c ~]$  sudo cat /etc/selinux/config | grep SELINUX
# SELINUX= can take one of these three values:
# NOTE: In earlier Fedora kernel builds, SELINUX=disabled would also
SELINUX=permissive
# SELINUXTYPE= can take one of these three values:
SELINUXTYPE=targeted
```

* Configurer le firewalld

Restreindre les ports d'écoute

```bash
sudo firewall-cmd --set-default-zone=drop
```

Ajouter des règles pour ssh et http (source adresse = ip du serveur proxy)

```bash
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="0.0.0.0/0" port port="6767" protocol="tcp" accept'
- sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="162.19.65.43" port port="80" protocol="tcp" accept'
```

* Configurer la conf sshd pour écouter sur un port différent de 22

```bash
[rocky@vps-d0e86d3c ~]$ sudo cat /etc/ssh/sshd_config | grep Port
Port {port souhaité}
```

* Installer et configurer Fail2Ban

```powershell
sudo dnf install epel-release

sudo dnf install fail2ban fail2ban-firewalld

sudo systemctl enable fail2ban

sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local

sudo vim /etc/fail2ban/jail.local

sudo systemctl restart fail2ban

```

* Installation du service Jirafeau

```bash
sudo dnf install php

sudo dnf install git 

```

Cloner le repo dans /var/www

```bash

git clone https://gitlab.com/mojo42/Jirafeau.git

cp /Jirafeau/lib/config.original.php /Jirafeau/lib/config.local.php
```

Modifier le fichier de config si besoin

```bash
sudo vim /var/www/Jirafeau/lib/config.local.php 
```

Donner la propriété et les permissions à l'utilisateur démarrant le serveur web (généralement apache)

```bash
sudo chown -R apache Jirafeau
```

* Installation et configuration du serveur web

**Installation**

```bash
sudo dnf install nginx
```

**Configuration**

```bash
[rocky@vps-d0e86d3c ~]$ cat /etc/nginx/nginx.conf | grep -A 20 "server {"
    server {
        listen       80;
        listen       [::]:80;
        server_name  _;
        root         /var/www/Jirafeau;
	index index.php;


        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        error_page 404 /404.html;
        location = /404.html {
        }

        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
        }
    }
```









