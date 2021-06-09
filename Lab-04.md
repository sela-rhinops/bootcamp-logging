# Lab 03: Deploy Filebeat using Docker Compose

## Tasks

 - Create Filebeat configuration file
 - Deploy Filebeat using Docker Compose
 - Deploy demo application
 - Visualize logs

## Create Filebeat configuration file
1. Create a dedicated folder for your Filebeat files in the workspace that we configured in the previous lab
```
mkdir ~/logging-lab/filebeat
```

2. Now let's create a basic configuration file for Filebeat (I will use vim, you can use a different editor)
```
vim ~/logging-lab/filebeat/filebeat.yml
```

3. The content of the file should be the below:
```
---
filebeat.config:
  modules:
    path: ${path.config}/modules.d/*.yml
    reload.enabled: false

filebeat.autodiscover:
  providers:
    - type: docker
      hints.enabled: true

output.elasticsearch:
  hosts: http://elasticsearch:9200

logging.level: info
logging.to_stderr: true
```

## Deploy Filebeat using Docker Compose

1. Let's add to the docker-compose file that we created in the previous lab a new service for filebeat
```
vim ~/logging-lab/docker-compose.yml
```

2. The content of the docker-compose file should be the following:
```
version: '3.2'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.12.1
    volumes:
      - type: bind
        source: ./elasticsearch/elasticsearch.yml
        target: /usr/share/elasticsearch/config/elasticsearch.yml
        read_only: true
      - type: volume
        source: elasticsearch
        target: /usr/share/elasticsearch/data
    ports:
      - "9200:9200"
      - "9300:9300"
    environment:
      ES_JAVA_OPTS: "-Xmx256m -Xms256m"
      ELASTIC_PASSWORD: changeme
      # Use single node discovery in order to disable production mode and avoid bootstrap checks.
      # see: https://www.elastic.co/guide/en/elasticsearch/reference/current/bootstrap-checks.html
      discovery.type: single-node
    networks:
      - elk
  kibana:
    image: docker.elastic.co/kibana/kibana:7.12.1
    volumes:
      - type: bind
        source: ./kibana/kibana.yml
        target: /usr/share/kibana/config/kibana.yml
        read_only: true
    ports:
      - "5601:5601"
    networks:
      - elk
    depends_on:
      - elasticsearch   
  filebeat:
    image: docker.elastic.co/beats/filebeat:7.12.1
    user: root
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker:/var/lib/docker
      - filebeat:/usr/share/filebeat/data:rw
      - ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml
    command: ["--strict.perms=false"]
    networks:
    - elk
    depends_on:
      - logstash
      - elasticsearch
networks:
  elk:
volumes:
  elasticsearch:
  filebeat:
```

3. Deploy Filebeat using docker compose
```
cd ~/logging-lab
docker-compose up -d
```

## Deploy demo application

1. Let's deploy the the demo "Guestbook" application
```
cd ../demo-app
docker-compose up -d
```

2. Go to localhost (port 80) and add some entries in the Guestbook

## Visualize logs

1. Now that we deployed filebeat we can go to Kibana and start visualizing our logs. In Kibana, go to Management > Stack Management > Data > Index Management. Here we can see the new index created by filebeat.

  ![filebeat-1](/images/filebeat-1.png)

2. Let's create an index pattern to visualize filebeat logs. You can use the "filebeat-*" pattern to match all filebeat indexes.

  ![filebeat-2](/images/filebeat-2.png)

3. Chose the field: @timestamp as Time field:

  ![filebeat-3](/images/filebeat-3.png)

4. Let's go to Analyze > Discover and chose the filebeat-* index pattern to see the logs being uploaded by filebeat.

  ![filebeat-4](/images/filebeat-4.png)

5. Add the following filter to see only logs from the Guestbook application services:

  ![filebeat-5](/images/filebeat-5.png)



