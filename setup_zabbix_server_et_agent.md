# TP1 Installation zabbix

## Installer zabbix serveur sur une VM linux



#### 1. Install and configure Zabbix for your platform

$ ` sudo -s`

$ ``wget https://repo.zabbix.com/zabbix/7.4/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.4+ubuntu24.04_all.deb``

$ ``dpkg -i zabbix-release_latest_7.4+ubuntu24.04_all.deb``

$ ``apt update``

###### Install Zabbix server, frontend, agent

$ ``apt install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent``



## 2. Configurer la base de données MySQL/PostgreSQL  

##### (téléchargé mariaDB plutot que my sql)

###### Create initial database

$ `sudo apt install mysql-server`

$ `mysql -uroot -p`
###### *enter -> password*

```sql
mysql> create database zabbix character set utf8mb4 collate utf8mb4_bin;
mysql> create user zabbix@localhost identified by 'password';
mysql> grant all privileges on zabbix.* to zabbix@localhost;
mysql> set global log_bin_trust_function_creators = 1;
mysql> quit;
```

###### -*Nous pourrons par la suite entrer le nouveau mot de passe de la base de données*-



###### Importer le schéma zabbix pour sql avec les données pour qu'elles fonctionnent ensemble.
$ ``zcat /usr/share/zabbix/sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix ``

###### Après avoir importer le schéma de la base de données il nous faut désactiver l'option log_bin_trust_function_creators avec cette commande: 

$ `mysql -uroot -p`
###### *enter -> new password*

```sql
mysql> set global log_bin_trust_function_creators = 0;
mysql> quit; 
```



###### Il nous faut maintenant changer le fichier /etc/zabbix/zabbix_server.conf

$ `vi /etc/zabbix/zabbix_server.conf`

``DBPassword=password``

###### Allow ports

$ ``ufw allow 80``
$ ``ufw allow 10050``
$ ``ufw allow 10051``

###### Changement de langues

$ `sudo dpkg-reconfigure locales`

$ `sudo systemctl restart apache2`

###### Démarrer le serveur zabbix et le processus de l'agent et lancer les au démarrage du système

$ ``systemctl restart zabbix-server zabbix-agent apache2``
$ `` systemctl enable zabbix-server zabbix-agent apache2 ``



### 3. Open Zabbix UI web page

##### L'URL par defaut pour Zabbix UI quand on utilise apache web serveur est http://host/zabbix

###### Interface Graphique : 

```
Monitoring / Host : 
    Create (host en haut a droite)
    	Host name 	= web lab.local
		Templates 	= Template/OperatingSystems - Linux by Zabbix agent
		Host Groupe = Linux servers
        Interfaces	= add +
        	Agent: 172.16.16.164 (agent ip a) IP/DNS 10050
```



###### téléchargement de mariah DB (optional) 

$ `sudo apt mariadb-server`
###### start MariaDb:

$ `systemctl restart mariadb`

$ `systemctl enable mariadb`

###### make SQL secure (optional)
$ `mysql_secure_installation`

$ `iptables -L INPUT` 		*( liste les règles en place sur le système)*
$ `ufw status`  			*(liste les règles mais pas forcément toute)*



### 4. Agent Web (Install and configure Zabbix-agent2)

###### Become root user

$ `sudo -s`

###### Install Zabbix repository Documentation

$ ``wget https://repo.zabbix.com/zabbix/7.4/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.4+ubuntu24.04_all.deb``
$ ``dpkg -i zabbix-release_latest_7.4+ubuntu24.04_all.deb``
$ ``apt update``

###### Install Zabbix agent 2

``apt install zabbix-agent2``

###### Install Zabbix agent 2 plugins

$ ``apt install zabbix-agent2-plugin-mongodb zabbix-agent2-plugin-mssql zabbix-agent2-plugin-postgresql``

###### modification du fichier de configuration

$ `vi /etc/zabbix/zabbix_agent2.conf`

*search `/Server`*
`Server=172.16.16.162`

*search `/ServerActive`*
`ServerActive=172.16.16.162`

###### allow ports

$ `ufw allow 10050`

###### Start Zabbix agent 2 process and make it start at system boot.

$ ``systemctl restart zabbix-agent2``

$ ``systemctl enable zabbix-agent2``