# Projet_Final Mise en place d'un outil de visualisation

## 1. Installation zabbix et configuration de zabbix

Pour commencer nous allons nous mettre en root

`` sudo -s ``
Puis installer le repository de zabbix 

``wget https://repo.zabbix.com/zabbix/7.4/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.4+ubuntu24.04_all.deb``
``dpkg -i zabbix-release_latest_7.4+ubuntu24.04_all.deb``
``apt update``

###  Install Zabbix server, frontend, agent

`` apt install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent``

### Téléchargement de mysql 

``sudo apt install mysql-server``

### Création de la base de données initial 

``mysql -uroot -p``

``password``
``mysql> create database zabbix character set utf8mb4 collate utf8mb4_bin;``
``mysql> create user zabbix@localhost identified by 'password';``
``mysql> grant all privileges on zabbix.* to zabbix@localhost;``
``mysql> set global log_bin_trust_function_creators = 1;``
``mysql> quit; ``

### 

``zcat /usr/share/zabbix/sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix ``

### Après avoir importer le schéma de la base de données il nous faut désactiver l'option log_bin_trust_function_creators

mysql -uroot -p
password
mysql> set global log_bin_trust_function_creators = 0;
mysql> quit; 

### Configurer la base de données pour zabbix 

dans les fichier /etc/zabbix/zabbix_server.conf 

``vim /etc/zabbix/zabbix_server.conf ``
``DBPassword=password``

### Autoriser les ports utiles

``ufw allow 80``
``ufw allow 10050``
``ufw allow 10051``

### Téléchargement de la langue en local 

``sudo dpkg-reconfigure locales``

``sudo systemctl restart apache2``

### Demarrer les processus zabbix-serveur et zabbix-agents 

``systemctl restart zabbix-server zabbix-agent apache2``
``systemctl enable zabbix-server zabbix-agent apache2 ``

### Ouvrir la page web de zabbix 

http://host(ip de la machine)/zabbix

# Partie web 

###### Install Zabbix agent2

 ``wget https://repo.zabbix.com/zabbix/7.4/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.4+ubuntu24.04_all.deb``
 ``dpkg -i zabbix-release_latest_7.4+ubuntu24.04_all.deb``
 ``apt update``

###### Install Zabbix agent 2

``apt install zabbix-agent2``

###### Install Zabbix agent 2 plugins

``apt install zabbix-agent2-plugin-mongodb zabbix-agent2-plugin-mssql zabbix-agent2-plugin-postgresql``

###### modification du fichier de configuration

`vi /etc/zabbix/zabbix_agent2.conf`

search `/Server`*
`Server=172.16.16.162`

search `/ServerActive`
``ServerActive=172.16.16.162``

###### allow ports

``ufw allow 10050``

###### Start Zabbix agent 2 process and make it start at system boot.

``systemctl restart zabbix-agent2``

``systemctl enable zabbix-agent2``

# Partie Windows

Après avoir crée un nouvel host 

Nous avons installer l'agent 2 

Et l'avons configurer

Nous avons par la suite créer une alerte 