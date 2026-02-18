# Installation de la stack ELK avec docker

### Installation docker

Debian Trixie 13 (stable)

au cas ou ces programmes sont installer :

``sudo apt remove $(dpkg --get-selections docker.io docker-compose docker-doc podman-docker containerd runc | cut -f1)``

apt might report that you have none of these packages installed

`` sudo apt update``

`` sudo apt install ca-certificates curl``

`` sudo install -m 0755 -d /etc/apt/keyrings``

`` sudo curl -fsSL https://download.docker.com/linux/debian/``

`` gpg -o /etc/apt/keyrings/docker.asc``

`` sudo chmod a+r /etc/apt/keyrings/docker.asc``

#### Add the repository to Apt sources:

``sudo tee /etc/apt/sources.list.d/docker.sources <<EOF``

``Types: deb``

``URIs: https://download.docker.com/linux/debian``

``Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")``

``Components: stable``

``Signed-By: /etc/apt/keyrings/docker.asc``

``EOF``
#### To install the latest version, run:

``sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-c``

#### The Docker service starts automatically after installation. To verify that Docker is running, use:

``sudo systemctl status docker``

#### Some systems may have this behavior disabled and will require a manual start:

`` sudo systemctl start docker``

#### Verify that the installation is successful by running the hello-world image:

``sudo docker run hello-world``

## Installation de elasticsearch

### First, let's start by defining the outline of our file structure.

├── .env

├── docker-compose.yml

├── filebeat.yml

├── logstash.conf

└── metricbeat.yml

### Environment file

Next, we’ll define variables to pass to the docker-compose via the .env file. These parameters will help us establish ports, memory limits, component versions, etc.

### .env

Project namespace (defaults to the current folder name if not set) 
COMPOSE_PROJECT_NAME=myproject

#### Password for the 'elastic' user (at least 6 characters)
ELASTIC_PASSWORD=changeme


#### Password for the 'kibana_system' user (at least 6 characters)
KIBANA_PASSWORD=changeme


#### Version of Elastic products
STACK_VERSION=8.7.1


#### Set the cluster name
CLUSTER_NAME=docker-cluster


#### Set to 'basic' or 'trial' to automatically start the 30-day trial
LICENSE=basic
LICENSE=trial


#### Port to expose Elasticsearch HTTP API to the host
ES_PORT=9200


#### Port to expose Kibana to the host
KIBANA_PORT=5601

## Setup and Elasticsearch node

### docker-compose.yml (‘setup’ container)
```docker

version: "3.8"


volumes:
 certs:
   driver: local
 esdata01:
   driver: local
 kibanadata:
   driver: local
 metricbeatdata01:
   driver: local
 filebeatdata01:
   driver: local
 logstashdata01:
   driver: local


networks:
 default:
   name: elastic
   external: false


services:
 setup:
   image: docker.elastic.co/elasticsearch/elasticsearch:${STACK_VERSION}
   volumes:
     - certs:/usr/share/elasticsearch/config/certs
   user: "0"
   command: >
     bash -c '
       if [ x${ELASTIC_PASSWORD} == x ]; then
         echo "Set the ELASTIC_PASSWORD environment variable in the .env file";
         exit 1;
       elif [ x${KIBANA_PASSWORD} == x ]; then
         echo "Set the KIBANA_PASSWORD environment variable in the .env file";
         exit 1;
       fi;
       if [ ! -f config/certs/ca.zip ]; then
         echo "Creating CA";
         bin/elasticsearch-certutil ca --silent --pem -out config/certs/ca.zip;
         unzip config/certs/ca.zip -d config/certs;
       fi;
       if [ ! -f config/certs/certs.zip ]; then
         echo "Creating certs";
         echo -ne \
         "instances:\n"\
         "  - name: es01\n"\
         "    dns:\n"\
         "      - es01\n"\
         "      - localhost\n"\
         "    ip:\n"\
         "      - 127.0.0.1\n"\
         "  - name: kibana\n"\
         "    dns:\n"\
         "      - kibana\n"\
         "      - localhost\n"\
         "    ip:\n"\
         "      - 127.0.0.1\n"\
         > config/certs/instances.yml;
         bin/elasticsearch-certutil cert --silent --pem -out config/certs/certs.zip --in config/certs/instances.yml --ca-cert config/certs/ca/ca.crt --ca-key config/certs/ca/ca.key;
         unzip config/certs/certs.zip -d config/certs;
       fi;
       echo "Setting file permissions"
       chown -R root:root config/certs;
       find . -type d -exec chmod 750 \{\} \;;
       find . -type f -exec chmod 640 \{\} \;;
       echo "Waiting for Elasticsearch availability";
       until curl -s --cacert config/certs/ca/ca.crt https://es01:9200 | grep -q "missing authentication credentials"; do sleep 30; done;
       echo "Setting kibana_system password";
       until curl -s -X POST --cacert config/certs/ca/ca.crt -u "elastic:${ELASTIC_PASSWORD}" -H "Content-Type: application/json" https://es01:9200/_security/user/kibana_system/_password -d "{\"password\":\"${KIBANA_PASSWORD}\"}" | grep -q "^{}"; do sleep 10; done;
       echo "All done!";
     '
   healthcheck:
     test: ["CMD-SHELL", "[ -f config/certs/es01/es01.crt ]"]
     interval: 1s
     timeout: 5s
     retries: 120
```
### docker-compose.yml (‘es01’ container)

```
 es01:
   depends_on:
     setup:
       condition: service_healthy
   image: docker.elastic.co/elasticsearch/elasticsearch:${STACK_VERSION}
   labels:
     co.elastic.logs/module: elasticsearch
   volumes:
     - certs:/usr/share/elasticsearch/config/certs
     - esdata01:/usr/share/elasticsearch/data
   ports:
     - ${ES_PORT}:9200
   environment:
     - node.name=es01
     - cluster.name=${CLUSTER_NAME}
     - discovery.type=single-node
     - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
     - bootstrap.memory_lock=true
     - xpack.security.enabled=true
     - xpack.security.http.ssl.enabled=true
     - xpack.security.http.ssl.key=certs/es01/es01.key
     - xpack.security.http.ssl.certificate=certs/es01/es01.crt
     - xpack.security.http.ssl.certificate_authorities=certs/ca/ca.crt
     - xpack.security.transport.ssl.enabled=true
     - xpack.security.transport.ssl.key=certs/es01/es01.key
     - xpack.security.transport.ssl.certificate=certs/es01/es01.crt
     - xpack.security.transport.ssl.certificate_authorities=certs/ca/ca.crt
     - xpack.security.transport.ssl.verification_mode=certificate
     - xpack.license.self_generated.type=${LICENSE}
   mem_limit: ${ES_MEM_LIMIT}
   ulimits:
     memlock:
       soft: -1
       hard: -1
   healthcheck:
     test:
       [
         "CMD-SHELL",
         "curl -s --cacert config/certs/ca/ca.crt https://localhost:9200 | grep -q 'missing authentication credentials'",
       ]
     interval: 10s
     timeout: 10s
     retries: 120
```

### Docker Compose tips

You will want to run all these commands in a terminal while in the same folder in which your docker-compose.yml file resides. My example folder:

`` docker compose up``

Si nous rencontrons cette erreur :
``{"@timestamp":"2023-04-14T13:16:22.148Z", "log.level":"ERROR", "message":"node validation exception\n[1] bootstrap checks failed. You must address the points described in the following [1] lines before starting Elasticsearch.\nbootstrap check failure [1] of [1]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]", "ecs.version": "1.2.0","service.name":"ES_ECS","event.dataset":"elasticsearch.server","process.thread.name":"main","log.logger":"org.elasticsearch.bootstrap.Elasticsearch","elasticsearch.node.name":"es01","elasticsearch.cluster.name":"docker-cluster"}``

The key takeaway here is max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144].

Ultimately, the command ``sysctl -w vm.max_map_count=262144`` needs to be run where the containers are being hosted.

Once complete, you can reboot Docker Desktop and retry your docker-compose up command.

Remember, the Setup container will exit on purpose after it has completed generating the certs and passwords.

So far so good, but let's test. 

We can use a command to copy the ca.crt out of the es01-1 container. Remember, the name of the set of containers is based on the folder from which the docker-compose.yml is running. For example, my directory is “elasticstack_docker” therefore, my command would look like this, based on the screenshot above:

docker cp

elasticstack_docker-es01-1:/usr/share/elasticsearch/config/certs/ca/ca.crt /tmp/.

Once the certificate is downloaded, run a curl command to query the Elasticsearch node:

curl --cacert /tmp/ca.crt -u elastic:changeme https://localhost:9200

Success!

Notice that we’re accessing Elasticsearch using localhost:9200. This is thanks to the port, which has been specified via the ports section of docker-compose.yml. This setting maps ports on the container to ports on the host and allows traffic to pass through your machine and into the docker container with that port specified.

## Installation de Kibana

For the Kibana config, we will utilize the certificate output from earlier. We will also specify that this node doesn't start until it sees that the Elasticsearch node above is up and running correctly.


### docker-compose.yml (‘kibana’ container)

```
kibana:
   depends_on:
     es01:
       condition: service_healthy
   image: docker.elastic.co/kibana/kibana:${STACK_VERSION}
   labels:
     co.elastic.logs/module: kibana
   volumes:
     - certs:/usr/share/kibana/config/certs
     - kibanadata:/usr/share/kibana/data
   ports:
     - ${KIBANA_PORT}:5601
   environment:
     - SERVERNAME=kibana
     - ELASTICSEARCH_HOSTS=https://es01:9200
     - ELASTICSEARCH_USERNAME=kibana_system
     - ELASTICSEARCH_PASSWORD=${KIBANA_PASSWORD}
     - ELASTICSEARCH_SSL_CERTIFICATEAUTHORITIES=config/certs/ca/ca.crt
     - XPACK_SECURITY_ENCRYPTIONKEY=${ENCRYPTION_KEY}
     - XPACK_ENCRYPTEDSAVEDOBJECTS_ENCRYPTIONKEY=${ENCRYPTION_KEY}
     - XPACK_REPORTING_ENCRYPTIONKEY=${ENCRYPTION_KEY}
   mem_limit: ${KB_MEM_LIMIT}
   healthcheck:
     test:
       [
         "CMD-SHELL",
         "curl -s -I http://localhost:5601 | grep -q 'HTTP/1.1 302 Found'",
       ]
     interval: 10s
     timeout: 10s
     retries: 120
```
Notice in our `environment` section that we’re specifying ELASTICSEARCH_HOSTS=https://es01:9200 We’re able to specify the container name here for our ES01 Elasticsearch container since we’re utilizing the Docker default networking. All containers that are using the “elastic” network that was specified at the beginning of our docker-compose.yml file will be able to properly resolve other container names and communicate with each other.

Let's load up Kibana and see if we can access it.

Consigne 
Monitoré avec lockstash les connexion ssh sur elk et faire remonté dans kibana

# Installation FileBeat (déja installer pour nous)

### Configuration FileBeat

Le fichier de configuration de Filebeat se retrouve dans /etc/filebeat/filebeat.yml

``` output.elasticsearch:
  hosts: ["localhost:9200"]
  username: "" #(si pas de login/mot de passe ne rien mettre)
  password: "" #(si pas de login/mot de passe ne rien mettre)

...

setup.kibana:
  host: "localhost:5601"

...

# Optionnelle
filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: true
  reload.period: 15s
```
### Pour initialiser le service à chaque démarrage de la machine, lancez la commande suivante:

`` sudo systemctl enable filebeat ``

type :filestream

fichier /etc/filebeat/filebeat.yml : 

rajouter dans docker compose (de logtach) que on attend de logstach de faire quelque chose (on doit le surcharger)

copie template dans host 
Appliquer en interface graphique template sur web.lab.local

https://git.zabbix.com/projects/ZBX/repos/zabbix/browse/templates/app/ssh_service/template_app_ssh_service.yaml?at=release%2F6.2


