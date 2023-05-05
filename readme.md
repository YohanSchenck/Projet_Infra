# Instruction d'installation du WAF

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

# Configuration SSH 

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
Cela vous permettera de ne pas vous prendre les attaque classic de spam en port 22, que seul vous et vos collaborateur puissier vous connecter seulement par clé ,que tous personnes inactive soit kick, de ne pas se connecter en root mais seulement par votre propre compte utilisateur et d'avoir une jolie banner quand on ce connecter.


# Configuration Fail2Ban


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

# Avoir acces au log et au regle de ModSecurity

* Fichier de log ModSecurity 

```bash 
sudo vim /var/log/modsec_audit.log
```

* Dossier des regles de ModSecurity 

```bash
sudo vim /usr/local/modsecurity-crs/rules/
```
