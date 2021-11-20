# Lab 01: Deploy Elasticsearch using the ECK Operator

## Tasks

 - Configure your workspace
 - Create Elasticsearch configuration file
 - Deploy Elasticsearch using CRDs
 - Explore Elasticsearch

## Configure your workspace

1. During the tutorial we will create several files that will be used to deploy each of the components of the Elastic Stack. To make it easier let's create a new directory to be used as our workspace (we will use "~/logging-lab" but you can use a different one if you prefer)

```
mkdir ~/logging-lab
```

2. Create a dedicated folder for your elasticsearch files

```
mkdir ~/logging-lab/elasticsearch
```

## Create Elasticsearch configuration file

1. Now let's create a Kubernetes configuration file for Elasticsearch (I will use vim, you can use a different editor)

```
vim ~/logging-lab/elasticsearch/elasticsearch.yml
```

2. The content of the file should be the below:

```
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: quickstart
spec:
  version: 7.14.0
  nodeSets:
  - name: default
    count: 1
    config:
      node.store.allow_mmap: false
```
As you may have noticed the "kind" of this object is Elasticsearch. This is one of the Custom Resource Definitions (CRD) that were installed as part of the operator's installation

## Deploy Elasticsearch using CRDs

1. Use kubectl to deploy the cluster with the configuration file created in the previous step.
  ```
  kubectl apply -f ~/logging-lab/elasticsearch/elasticsearch.yml -n logging
  ```

2. It may take up to a few minutes until all the resources are created and the cluster is ready for use. You can see the status of the pods with:
  ```
  kubectl get po -w -n logging
  ```

3. You can also get an overview of the current Elasticsearch clusters in the Kubernetes cluster, including health, version and number of nodes by:
  ```
  kubectl get elasticsearch -n logging
  ```
  When you create the cluster, there is no HEALTH status and the PHASE is empty. After a while, the PHASE turns into Ready, and HEALTH becomes green. The HEALTH status comes from Elasticsearch’s cluster health API.

4. Access the logs of the Elasticsearch Pod:
  ```
  kubectl logs -f quickstart-es-default-0 -n logging
  ```

5. Lets change the service that was created for the cluster from ClusterIp to NodePort to have access to it.
  ```
  kubectl patch svc quickstart-es-http -n logging --type='json' -p '[{"op":"replace","path":"/spec/type","value":"NodePort"}]'
  ```

## Explore Elasticsearch

1. When creating an Elasticsearch cluster with the ECK operator it is secure by default. This means that it creates a self signed certificate (if it was not provided) and a secret containing the password for the elastic user (admin). We will need the password of this user to proceed.
  ```
  PASSWORD=$(kubectl get secret -n logging quickstart-es-elastic-user -o go-template='{{.data.elastic | base64decode}}')
  ```

2. Elasticsearch itself does not have a UI but we are going to do a couple of requests to it's API. The first request will be to the / endpoint to see if it was installed succesfully.
  ```
  curl -k -u elastic:$PASSWORD https://<NODE-IP>:<NODEPORT>
  ```

3. We can check the health of the cluster by sending a GET request to the health endpoint. We will see that the health of the cluster is yellow and this is because we are using a single node cluster.
  ```
    curl -k -u elastic:$PASSWORD https://<NODE-IP>:<NODEPORT>/_cat/health
  ```

4. We are going to create two indices where we will uploading our documents.
  ```
  curl -k -u elastic:$PASSWORD -XPUT "https://<NODE-IP>:<NODEPORT>/app-logs-auth"

  curl -k -u elastic:$PASSWORD -XPUT "https://<NODE-IP>:<NODEPORT>/app-logs-actions"
  ```
5. Now we are going to upload a document into each of the indices that we created.
  ```
  curl -k -u elastic:$PASSWORD -XPOST 'https://<NODE-IP>:<NODEPORT>/app-logs-auth/my_app' -H 'Content-Type: application/json' -d'
  {
  	"timestamp": "2020-01-24 12:34:56",
  	"message": "User logged in",
    "username": "mickeymouse",
  	"user_id": 4,
  	"admin": "false"
  }
  '
  
  curl -k -u elastic:$PASSWORD -XPOST 'https://<NODE-IP>:<NODEPORT>/app-logs-actions/my_app' -H 'Content-Type: application/json' -d'
  {
  	"timestamp": "2020-01-24 12:34:57",
  	"message": "User performed action",
    "username": "mickeymouse",  
    "action": "Buy",
    "ammount": 37,
  	"user_id": 999,
  	"admin": false
  }
  '
  ```
6. Indexed documents are available for search in near real-time. To search data, we are going to use the _search API. The following request will get all the documents in all the indexes that match the app-logs-* pattern.
  ```
  curl -k -u elastic:$PASSWORD -XGET 'https://<NODE-IP>:<NODEPORT>/app-logs-*/_search?pretty'
  ```

7. The following request uses the same _search api but this time we are sending a query that matches documents that contain the string "User logged in" in the field "message"
  ```
  curl -k -u elastic:$PASSWORD -XGET 'https://<NODE-IP>:<NODEPORT>/app-logs-*/_search?pretty' -H 'Content-Type: application/json' -d'
  {
    "query": {
      "match_phrase": {
        "message": "User logged in"
      }
    }
  }
  '
  ```






