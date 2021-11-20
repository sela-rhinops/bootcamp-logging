# Lab 03: Deploy Filebeat using the ECK Operator

## Tasks

 - Create Filebeat configuration file
 - Deploy Filebeat using the ECK Operator
 - Deploy demo application
 - Visualize logs

## Create Filebeat configuration file
1. Create a dedicated folder for your Filebeat files in the workspace that we configured in the previous lab
```
mkdir ~/logging-lab/filebeat
```

2. Now let's create a Kubernetes configuration file for Filebeat (I will use vim, you can use a different editor)
```
vim ~/logging-lab/filebeat/filebeat.yml
```

3. The content of the file should be the below:
```
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: quickstart
spec:
  type: filebeat
  version: 7.14.0
  elasticsearchRef:
    name: quickstart
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

## Deploy Filebeat using CRDs

1. Use kubectl to deploy filebeat with the configuration file created in the previous step.
  ```
  kubectl apply -f ~/logging-lab/filebeat/filebeat.yml -n logging
  ```

2. It may take up to a few minutes until all the resources are created and filebeat connects to Elasticsearch and Kibana. You can see the status of the pods with:
  ```
  kubectl get po -w
  ```

3. You can also get an overview of all beats running in the Kubernetes cluster and get information such as health, name of deployment, desired and current number of replicas, type of beat and version by:
  ```
  kubectl get beats -n logging
  ```

4. Access the logs of filebeat Pods:
  ```
  kubectl logs -f quickstart-beat-filebeat-<uuid>
  ```

5. List all the Pods belonging to a given Beat.
  ```
  kubectl get pods --selector='beat.k8s.elastic.co/name=quickstart-beat-filebeat'
  ```

## Deploy demo application

1. Let's deploy the the demo "Guestbook" application
```
kubectl apply -f ./demo-app
```

2. Go to localhost:<nodeport> and add some entries in the Guestbook

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



