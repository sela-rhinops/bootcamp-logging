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

```
---
replicas: 1

# Allows you to add any config files in /usr/share/logstash/config/
# such as logstash.yml and log4j2.properties
#
# Note that when overriding logstash.yml, `http.host: 0.0.0.0` should always be included
# to make default probes work.
logstashConfig: {}
#  logstash.yml: |
#    key:
#      nestedkey: value
#  log4j2.properties: |
#    key = value

# Allows you to add any pipeline files in /usr/share/logstash/pipeline/
### ***warn*** there is a hardcoded logstash.conf in the image, override it first
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
        password => <YOUR-ELASTIC-PASSWORD>
        index => "logstash-guestbook-%{+YYYY.MM.dd}"
        cacert => '/usr/share/logstash/ssl/ca.crt'
      }
  	}
    else {
      elasticsearch {
  	    hosts => "https://quickstart-es-http:9200"
        user => elastic
        password => <YOUR-ELASTIC-PASSWORD>
        index => "logstash-other-%{+YYYY.MM.dd}"
        cacert => '/usr/share/logstash/ssl/ca.crt'
      }
    }
  }

# Allows you to add any pattern files in your custom pattern dir
logstashPatternDir: "/usr/share/logstash/patterns/"
logstashPattern: {}
#    pattern.conf: |
#      DPKG_VERSION [-+~<>\.0-9a-zA-Z]+

# Extra environment variables to append to this nodeGroup
# This will be appended to the current 'env:' key. You can use any of the kubernetes env
# syntax here
extraEnvs: []
#  - name: MY_ENVIRONMENT_VAR
#    value: the_value_goes_here

# Allows you to load environment variables from kubernetes secret or config map
envFrom:
- secretRef:
    name: quickstart-es-elastic-user

# Add sensitive data to k8s secrets
secrets: []
#  - name: "env"
#    value:
#      ELASTICSEARCH_PASSWORD: "LS1CRUdJTiBgUFJJVkFURSB"
#      api_key: ui2CsdUadTiBasRJRkl9tvNnw
#  - name: "tls"
#    value:
#      ca.crt: |
#        LS0tLS1CRUdJT0K
#        LS0tLS1CRUdJT0K
#        LS0tLS1CRUdJT0K
#        LS0tLS1CRUdJT0K
#      cert.crt: "LS0tLS1CRUdJTiBlRJRklDQVRFLS0tLS0K"
#      cert.key.filepath: "secrets.crt" # The path to file should be relative to the `values.yaml` file.


# A list of secrets and their paths to mount inside the pod
secretMounts:
  - name: ca-certs
    path: /usr/share/logstash/ssl
    secretName: quickstart-es-http-certs-public

hostAliases: []
#- ip: "127.0.0.1"
#  hostnames:
#  - "foo.local"
#  - "bar.local"

image: "docker.elastic.co/logstash/logstash"
imageTag: "7.14.0"
imagePullPolicy: "IfNotPresent"
imagePullSecrets: []

podAnnotations: {}

# additionals labels
labels: {}

logstashJavaOpts: "-Xmx1g -Xms1g"

resources:
  requests:
    cpu: "100m"
    memory: "1536Mi"
  limits:
    cpu: "1000m"
    memory: "1536Mi"

volumeClaimTemplate:
  accessModes: [ "ReadWriteOnce" ]
  resources:
    requests:
      storage: 1Gi

rbac:
  create: false
  serviceAccountAnnotations: {}
  serviceAccountName: ""
  annotations: {}
    #annotation1: "value1"
    #annotation2: "value2"
    #annotation3: "value3"

podSecurityPolicy:
  create: false
  name: ""
  spec:
    privileged: false
    fsGroup:
      rule: RunAsAny
    runAsUser:
      rule: RunAsAny
    seLinux:
      rule: RunAsAny
    supplementalGroups:
      rule: RunAsAny
    volumes:
      - secret
      - configMap
      - persistentVolumeClaim

persistence:
  enabled: false
  annotations: {}

extraVolumes: ""
  # - name: extras
  #   emptyDir: {}

extraVolumeMounts: ""
  # - name: extras
  #   mountPath: /usr/share/extras
  #   readOnly: true

extraContainers: ""
  # - name: do-something
  #   image: busybox
  #   command: ['do', 'something']

extraInitContainers: ""
  # - name: do-something
  #   image: busybox
  #   command: ['do', 'something']

# This is the PriorityClass settings as defined in
# https://kubernetes.io/docs/concepts/configuration/pod-priority-preemption/#priorityclass
priorityClassName: ""

# By default this will make sure two pods don't end up on the same node
# Changing this to a region would allow you to spread pods across regions
antiAffinityTopologyKey: "kubernetes.io/hostname"

# Hard means that by default pods will only be scheduled if there are enough nodes for them
# and that they will never end up on the same node. Setting this to soft will do this "best effort"
antiAffinity: "hard"

# This is the node affinity settings as defined in
# https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#node-affinity-beta-feature
nodeAffinity: {}

# The default is to deploy all pods serially. By setting this to parallel all pods are started at
# the same time when bootstrapping the cluster
podManagementPolicy: "Parallel"

httpPort: 9600

# Custom ports to add to logstash
extraPorts: []
  # - name: beats
  #   containerPort: 5001

updateStrategy: RollingUpdate

# This is the max unavailable setting for the pod disruption budget
# The default value of 1 will make sure that kubernetes won't allow more than 1
# of your pods to be unavailable during maintenance
maxUnavailable: 1

podSecurityContext:
  fsGroup: 1000
  runAsUser: 1000

securityContext:
  capabilities:
    drop:
    - ALL
  # readOnlyRootFilesystem: true
  runAsNonRoot: true
  runAsUser: 1000

# How long to wait for logstash to stop gracefully
terminationGracePeriod: 120

# Probes
# Default probes are using `httpGet` which requires that `http.host: 0.0.0.0` is part of
# `logstash.yml`. If needed probes can be disabled or overrided using the following syntaxes:
#
# disable livenessProbe
# livenessProbe: null
#
# replace httpGet default readinessProbe by some exec probe
# readinessProbe:
#   httpGet: null
#   exec:
#     command:
#       - curl
#      - localhost:9600

livenessProbe:
  httpGet:
    path: /
    port: http
  initialDelaySeconds: 300
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3
  successThreshold: 1

readinessProbe:
  httpGet:
    path: /
    port: http
  initialDelaySeconds: 60
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3
  successThreshold: 3

## Use an alternate scheduler.
## ref: https://kubernetes.io/docs/tasks/administer-cluster/configure-multiple-schedulers/
##
schedulerName: ""

nodeSelector: {}
tolerations: []

nameOverride: ""
fullnameOverride: ""

lifecycle: {}
  # preStop:
  #   exec:
  #     command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /usr/share/message"]
  # postStart:
  #   exec:
  #     command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /usr/share/message"]

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
#  annotations: {}
#  hosts:
#    - host: logstash.local
#      paths:
#        - path: /logs
#          servicePort: 8080
#  tls: []
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
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: filebeat
subjects:
- kind: ServiceAccount
  name: filebeat
  namespace: default
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
kubectl apply ~/logging-lab/filebeat/ -n logging
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

