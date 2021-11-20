# Lab 01: Deploy The ECK Operator

## Tasks

 - Deploy the Operator
 - Troubleshooting

## Deploy the Operator

1. First lets add the ECK operator's chart to your helm charts
  ```
  helm repo add elastic https://helm.elastic.co
  helm repo update
  ```
2. Now we are going to deploy the Operator. With the deployment of the Operator we also get new "Building Blocks" called CRD's. We will be using these CRD's to deploy the Elastic Stack and the operator will make sure that that the cluster is always in the desired state. We will be using helm to deploy the operator but this can be done via the official ![all-in-one.yaml](https://download.elastic.co/downloads/eck/1.6.0/all-in-one.yaml) file.
  ```
  helm install elastic-operator elastic/eck-operator -n elastic-system --create-namespace
  ```
  This is the default mode of installation and is equivalent to installing ECK using the all-in-one.yaml file.

2. After deploying the operator wait until the Operator's Pod is up and Running
  ```
  kubectl get po -n elastic-system -w
  ```

## Troubleshooting

1. To view the operator's logs use the following commmand:
  ```
  kubectl -n elastic-system logs -f statefulset.apps/elastic-operator
  ```

2. When the default logs are not enough to troubleshoot it is possible to change the log level of the operator's pod by running:
  ```
  kubectl edit statefulset.apps -n elastic-system elastic-operator
  ```

3. Then change the args array as follows:
  ```
    spec:
    containers:
    - args:
      - manager
      - --log-verbosity=1
  ```

4. For debugging purposes, you might want to temporarily prevent ECK from modifying Kubernetes resources belonging to a particular Elastic Stack resource. To do this, annotate the Elastic object with eck.k8s.elastic.co/managed=false. This annotation can be added to any of the following types of objects:
  - Elasticsearch
  - Kibana
  - ApmServer
  ```
  metadata:
  annotations:
    eck.k8s.elastic.co/managed: "false"
  ```
