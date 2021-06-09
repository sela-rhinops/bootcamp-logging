# Lab 04: Deploy Logstash using Docker Compose

## Tasks

 - Create Logstash configuration file
 - Change Filebeat configuration file
 - Deploy Logstash using Docker Compose
 - Visualize logs

## Create Logstash configuration file

1. Create a dedicated folder for your logstash files in the workspace that we configured in the previous lab
```
mkdir ~/logging-lab/logstash
```

2. Now let's create a basic configuration file for logstash (I will use vim, you can use a different editor)

```
vim ~/logging-lab/logstash/logstash.conf
```
3. The content of the file should be the below:

```
input {
	beats {
		port => 5044
	}
}

filter {
  if [docker][container][labels][com_docker_compose_service] == "frontend" {
    grok {
      match => { "message" => "%{COMBINEDAPACHELOG}+%{GREEDYDATA:extra_fields}" }
      overwrite => [ "message" ]
    }
    date {
      match => [ "timestamp" , "dd/MMM/yyyy:HH:mm:ss Z" ]
    }
    geoip {
      source => "clientip"
    }
  }
}

output {
  if [docker][container][labels][app] == "guestbook" {
	  elasticsearch {
	  	hosts => "elasticsearch:9200"
      index => "logstash-guestbook-%{+YYYY.MM.dd}"
    }
	}
  else {
    	elasticsearch {
	  	hosts => "elasticsearch:9200"
      index => "logstash-other-%{+YYYY.MM.dd}"
    }
  }
}
```
4. Let's configure filebeat's output to send logs to logstash and not directly to elasticsearch
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

output.logstash:
  hosts: logstash:5044

logging.level: info
logging.to_stderr: true
```

## Deploy Logstash using Docker Compose

1. Let's add to the docker-compose file that we created in the previous lab a new service for logstash

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
      - ./elasticsearch/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml:ro
      - elasticsearch:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
      - "9300:9300"
    environment:
      ES_JAVA_OPTS: "-Xmx256m -Xms256m"
      ELASTIC_PASSWORD: changeme
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
  logstash:
    image: docker.elastic.co/logstash/logstash:7.12.1
    volumes:
      - ./logstash/logstash.conf:/usr/share/logstash/pipeline/logstash.conf:ro
    ports:
      - "5044:5044"
    environment:
      LS_JAVA_OPTS: "-Xmx256m -Xms256m"
    networks:
      - elk
    depends_on:
      - elasticsearch
networks:
  elk:
volumes:
  elasticsearch:
  filebeat:
```

## Visualize logs

1. Now that we deployed logstash we can go to Kibana and start visualizing our logs. In Kibana, go to Management > Stack Management > Data > Index Management. Here we can see the new indexes created by logstash, one for the demo application and the other one for all the other logs.

  ![logstash-1](/images/logstash-1.png)

2. Let's create two index patterns to visualize them. You can use the "logstash-guestbook-*" and "logstash-other-*" patterns to match all indexes that were created by our logstash pipeline.

  ![logstash-2](/images/logstash-2.png)
  *Chose the field: @timestamp as Time field*

3. Let's go to Analyze > Discover and choose the logstash-other-* index pattern to see the logs being uploaded by logstash.

  ![logstash-3](/images/logstash-3.png)

4. Generate some load in the guestbook application

5. If we inspect the logs in the logstash-guestbook-* pattern we will see that logstash has parsed the logs and enriched all frontend service's records with more data.

  ![logstash-4](/images/logstash-4.png)

