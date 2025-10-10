# Lancer le conteneur Ubuntu
```
docker run -it -d \
  --name zabbix-ubuntu \
  -p 8080:80 \
  -p 10051:10051 \
  ubuntu:22.04
```
```
docker ps
```
```
docker exec -it zabbix-ubuntu bash
```
```
apt update && apt install -y wget curl gnupg2 software-properties-common lsb-release sudo \
mariadb-server mariadb-client apache2 php php-mysql php-gd php-bcmath \
php-ldap php-mbstring php-xml php-json php-zip locales nano
```

# Configurer locale et timezone cohérentes
```
locale-gen en_US.UTF-8
```
```
update-locale LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8
```

'# locale-gen fr_FR.UTF-8 </br>
'# update-locale LANG=fr_FR.UTF-8 LC_ALL=fr_FR.UTF-8 </br>

# Définir la timezone pour le conteneur et PHP
```
ln -sf /usr/share/zoneinfo/Europe/Paris /etc/localtime
```
```
echo "Europe/Paris" > /etc/timezone
```
```
dpkg-reconfigure -f noninteractive tzdata
```
# Vérifier les dates
```
locale
```
```
date
```
```
php -r "echo date('Y-m-d H:i:s');"
```


# Initialiser MySQL : Lancer MySQL en arrière-plan (pas de systemctl/service dans Docker)
```
service mariadb start
```
'# mysqld_safe &


```
mysql -u root -p <<EOF
CREATE DATABASE IF NOT EXISTS zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
USE zabbix;
CREATE USER IF NOT EXISTS 'zabbix'@'localhost' IDENTIFIED BY 'zabbixpass';
CREATE USER IF NOT EXISTS 'zabbix'@'127.0.0.1' IDENTIFIED BY 'zabbixpass';
GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';
GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'127.0.0.1';
FLUSH PRIVILEGES;
EOF
```
# Ajouter le dépôt Zabbix et installer packages
```
wget https://repo.zabbix.com/zabbix/6.4/ubuntu/pool/main/z/zabbix-release/zabbix-release_6.4-1+ubuntu22.04_all.deb
```
```
dpkg -i zabbix-release_6.4-1+ubuntu22.04_all.deb
```
```
apt update
```
```
apt install -y zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf \
zabbix-agent zabbix-sql-scripts
```
# Importer le schéma Zabbix
```
dpkg -L zabbix-sql-scripts | grep \.sql
```
```
zcat /usr/share/doc/zabbix-sql-scripts/mysql/create.sql.gz | mysql -u zabbix -p zabbixpass zabbix
```
```
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql -u zabbix -pzabbixpass zabbix
```
'#zcat /usr/share/zabbix-sql-scripts/mysql/images.sql.gz | mysql -u zabbix -pzabbixpass zabbix
'#zcat /usr/share/zabbix-sql-scripts/mysql/data.sql.gz   | mysql -u zabbix -pzabbixpass zabbix

# Verification
```
mysql -u root -p 
```

```
USE zabbix;
```
```
SHOW TABLES;
```
```
SELECT * FROM dbversion;
```
# Configurer Zabbix Server
Éditer /etc/zabbix/zabbix_server.conf
```
DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=zabbixpass
```
# Configurer Apache et PHP pour le frontend
Éditer /etc/zabbix/apache.conf :
```
php_value date.timezone Europe/Paris
```
# Ajouter le ServerName pour Apache :
```
echo "ServerName localhost" >> /etc/apache2/apache2.conf
```
# Configurer Zabbix Agent
Éditer /etc/zabbix/zabbix_agentd.conf :
```
Server=127.0.0.1
ServerActive=127.0.0.1
Hostname=ZabbixUbuntu
```
# Créer le répertoire manquant
```
mkdir -p /run/zabbix
```
```
chown zabbix:zabbix /run/zabbix
```
# Script de demarrage : 



Créer un script pour démarrer tous les services

```
service mariadb start
```
```
service mariadb status
```
```
service zabbix-server start
```
'# /usr/sbin/zabbix_server -c /etc/zabbix/zabbix_server.conf 
```
service zabbix-server status
```
```
service zabbix-agent start
```

'#/usr/sbin/zabbix_agentd -c /etc/zabbix/zabbix_agentd.conf &

```
service zabbix-agent status
```
# Vérifier les logs

Tapez ctrl+shift+T
```
docker exec -it zabbix-ubuntu tail -f /var/log/zabbix/zabbix_server.log
```
```
service apache2 start
```
'#
Apache tourne au premier plan, le conteneur reste actif. </br>
MySQL, Zabbix Server et Agent tournent en arrière-plan. </br>

# Accéder au frontend
Navigateur : http://localhost:8080/zabbix </br></br>
User: Admin </br>
Password: zabbix <br>

