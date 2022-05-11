# Lab 04: Deploy Logstash using the ECK Operator

## Tasks

 - Create Logstash configuration file
 - Change Filebeat configuration file
 - Deploy Logstash using the ECK Operator
 - Visualize logs

## Create Logstash configuration file

1. Create a dedicated folder for your logstash files in the workspace that we configured in the previous lab
```
mkdir ~/logging-lab/logstash
```

2. We will be using the official logstash helm chart but we need to make some modifications on the values so let's create a values.yaml file (I will use vim, you can use a different editor)

```
vim ~/logging-lab/logstash/values.yaml
```
3. The content of the file should be the below:
  - *NOTE: Remember change <ELASTICSEARCH_PASSWORD> to the actual password of your Elasticsearch Cluster's elastic user*

```
---
replicas: 1

logstashPipeline: 
 logstash.conf: |
  input {
  	beats {
  		port => 5044
  	}
  }
  filter {
    if [kubernetes][labels][app_kubernetes_io/name] == "guestbook" {
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
    if [kubernetes][labels][demo] == "guestbook-app" {
  	  elasticsearch {
  	  	hosts => "https://quickstart-es-http:9200"
        user => elastic
        password => <ELASTICSEARCH_PASSWORD>
        index => "logstash-guestbook-%{+YYYY.MM.dd}"
        cacert => '/usr/share/logstash/ssl/ca.crt'
      }
  	}
    else {
      elasticsearch {
  	    hosts => "https://quickstart-es-http:9200"
        user => elastic
        password => <ELASTICSEARCH_PASSWORD>
        index => "logstash-other-%{+YYYY.MM.dd}"
        cacert => '/usr/share/logstash/ssl/ca.crt'
      }
    }
  }

envFrom:
- secretRef:
    name: quickstart-es-elastic-user

secretMounts:
  - name: ca-certs
    path: /usr/share/logstash/ssl
    secretName: quickstart-es-http-certs-public

image: "docker.elastic.co/logstash/logstash"
imageTag: "7.14.0"

service: 
 annotations: {}
 type: ClusterIP
 ports:
   - name: beats
     port: 5044
     protocol: TCP
     targetPort: 5044

ingress:
  enabled: false
```

4. Let's configure filebeat's output to send logs to logstash and not directly to elasticsearch this can be done by using kubectl edit or changing the yaml file and then applying it.
```
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: quickstart
spec:
  type: filebeat
  version: 7.14.0
  kibanaRef:
    name: quickstart
  config:
    filebeat:
      autodiscover:
        providers:
        - type: kubernetes
          node: ${NODE_NAME}
          hints:
            enabled: true
            default_config:
              type: container
              paths:
              - /var/log/containers/*${data.kubernetes.container.id}.log
    processors:
    - add_cloud_metadata: {}
    - add_host_metadata: {}
    output.logstash.hosts: logstash-logstash
  daemonSet:
    podTemplate:
      spec:
        serviceAccountName: filebeat
        automountServiceAccountToken: true
        terminationGracePeriodSeconds: 30
        dnsPolicy: ClusterFirstWithHostNet
        hostNetwork: true # Allows to provide richer host metadata
        containers:
        - name: filebeat
          securityContext:
            runAsUser: 0
            # If using Red Hat OpenShift uncomment this:
            #privileged: true
          volumeMounts:
          - name: varlogcontainers
            mountPath: /var/log/containers
          - name: varlogpods
            mountPath: /var/log/pods
          - name: varlibdockercontainers
            mountPath: /var/lib/docker/containers
          env:
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
        volumes:
        - name: varlogcontainers
          hostPath:
            path: /var/log/containers
        - name: varlogpods
          hostPath:
            path: /var/log/pods
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: filebeat
rules:
- apiGroups: [""] # "" indicates the core API group
  resources:
  - namespaces
  - pods
  - nodes
  verbs:
  - get
  - watch
  - list
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: filebeat
  namespace: logging
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: filebeat
subjects:
- kind: ServiceAccount
  name: filebeat
  namespace: logging
roleRef:
  kind: ClusterRole
  name: filebeat
  apiGroup: rbac.authorization.k8s.io
```

## Deploy Logstash using Helm and update filebeat configuration

1. Add the logstash helm repository to your helm repos
```
helm repo add elastic https://helm.elastic.co
```

2. Install Logstash using Helm
```
helm install logstash elastic/logstash -n logging -f ~/logging-lab/logstash/values.yaml
```

3. Update Filebeat configuration to point to Logstash instead of directly Elasticsearch.
```
kubectl apply -f ~/logging-lab/filebeat/ -n logging
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

