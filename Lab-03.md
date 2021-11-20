# Lab 02: Deploy Kibana using the ECK Operator

## Tasks

 - Create Kibana configuration file
 - Deploy Kibana using CRDs
 - Explore Kibana

## Create Kibana configuration file

1. Create a dedicated folder for your Kibana files in the workspace that we configured in the previous lab

```
mkdir ~/logging-lab/Kibana
```


2. Now let's create a Kubernetes configuration file for Kibana (I will use vim, you can use a different editor)

```
vim ~/logging-lab/Kibana/kibana.yml
```

3. The content of the file should be the below:

```
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: quickstart
spec:
  version: 7.14.0
  count: 1
  elasticsearchRef:
    name: quickstart
```

## Deploy Kibana using CRDs

1. Now lets use kubectl to deploy kibana using the yaml file that we created in the previous step.
  ```
  kubectl apply -f ~/logging-lab/Kibana/kibana.yml -n logging
  ```

2. It may take up to a few minutes until all the resources are created and the cluster is ready for use. You can see the status of the pods with:
  ```
  kubectl get po -w -n logging
  ```

3. You can also get an overview of all Kibana instaces running in the Kubernetes cluster, including health, version and number of nodes by:
  ```
  kubectl get kibana -n logging
  ```

4. Access the logs of the Kibana Pod:
  ```
  kubectl logs -f quickstart-kb-<uuid>-<uuid> -n logging
  ```

5. Lets change the service that was created for Kibana from ClusterIp to NodePort to have access to it.
  ```
  kubectl patch svc quickstart-kb-http --type='json' -p '[{"op":"replace","path":"/spec/type","value":"NodePort"}]' -n logging
  ```

## Explore Kibana

1. First lets retrieve the credentials of the elastic user so that we can use them to login into kibana
  ```
  PASSWORD=$(kubectl get secret quickstart-es-elastic-user -n logging -o go-template='{{.data.elastic | base64decode}}') &&  echo $PASSWORD
  ```
  Copy the password from the terminal by selecting it and using ctrl+insert

1. Browse to your kibana instance from any browser
  ```
  https://<NODE-IP>:<NODEPORT>
  ```

2. Log In with the following credentials
```
username: elastic
password: <elastic-user-password>
```

3. Go to Management > Stack Management > Kibana > Index patterns

  ![index-pattern-1](/images/index-pattern-1.png)

4. Let's create a Kibana index pattern to visualize all the indexes that match the pattern. Click on "Create index pattern" 

  ![index-pattern-2](/images/index-pattern-2.png)

5. You can use the "app-logs-*" pattern to match the indexes that we created in the previous lab.

  ![index-pattern-3](/images/index-pattern-3.png)

6. Now let's go Analitycs > Discover to visualize all the documents that are present in the indices that match the index pattern that we created.

  ![discover](/images/discover.png)

7. Kibana comes with built in console that can help us write requests to elasticsearch in a more confortable way. To find it go to Management > Dev Tools

  ![dev-tools](/images/dev-tools.png)

8. Lets use the _bulk API to upload some more data. With the _bulk API you can perform many operations in a single request. We will use it to upload many documents at once to our app-logs-actions. Copy and paste the following into the console.

```
POST _bulk
{ "index":{ "_index": "app-logs-actions" } }
{ "timestamp": "2020-01-24 12:34:56", "message": "User performed action", "username": "mickeymouse", "action": "Buy", "ammount": 43, "user_id": 999, "admin": false}
{ "index":{ "_index": "app-logs-actions" } }
{ "timestamp": "2020-01-24 12:34:56", "message": "User performed action", "username": "jhon", "action": "Buy", "ammount": 67, "user_id": 999, "admin": false}
{ "index":{ "_index": "app-logs-actions" } }
{ "timestamp": "2020-01-24 12:34:56", "message": "User performed action", "username": "mickeymouse", "action": "Buy", "ammount": 12, "user_id": 999, "admin": false}
{ "index":{ "_index": "app-logs-actions" } }
{ "timestamp": "2020-01-24 12:34:56", "message": "User performed action", "username": "jhon", "action": "Buy", "ammount": 54, "user_id": 999, "admin": false}
{ "index":{ "_index": "app-logs-actions" } }
{ "timestamp": "2020-01-24 12:34:56", "message": "User performed action", "username": "mickeymouse", "action": "Buy", "ammount": 79, "user_id": 999, "admin": false}
{ "index":{ "_index": "app-logs-actions" } }
{ "timestamp": "2020-01-24 12:34:56", "message": "User performed action", "username": "fernando", "action": "Buy", "ammount": 34, "user_id": 999, "admin": false}
{ "index":{ "_index": "app-logs-actions" } }
{ "timestamp": "2020-01-24 12:34:56", "message": "User performed action", "username": "fernando", "action": "Buy", "ammount": 767, "user_id": 999, "admin": false}
{ "index":{ "_index": "app-logs-actions" } }
{ "timestamp": "2020-01-24 12:34:56", "message": "User performed action", "username": "fernando", "action": "Buy", "ammount": 978, "user_id": 999, "admin": false}
{ "index":{ "_index": "app-logs-actions" } }
{ "timestamp": "2020-01-24 12:34:56", "message": "User performed action", "username": "fernando", "action": "Buy", "ammount": 234, "user_id": 999, "admin": false}
{ "index":{ "_index": "app-logs-actions" } }
{ "timestamp": "2020-01-24 12:34:56", "message": "User performed action", "username": "mickeymouse", "action": "Buy", "ammount": 345, "user_id": 999, "admin": false}
{ "index":{ "_index": "app-logs-actions" } }
{ "timestamp": "2020-01-24 12:34:56", "message": "User performed action", "username": "mickeymouse", "action": "Buy", "ammount": 424, "user_id": 999, "admin": false}
{ "index":{ "_index": "app-logs-actions" } }
{ "timestamp": "2020-01-24 12:34:56", "message": "User performed action", "username": "jhon", "action": "Buy", "ammount": 123, "user_id": 999, "admin": false}
```

9. Click on the play button to perform the request and get the response.

  ![dev-tools](/images/dev-tools-2.png)

10. Now that we have some data lets create our first dashboard. Go to Analytics > Dashboard and click the "Create New Dashboard" Button

  ![dashboard-1](/images/dashboard-1.png)

11. Click on the create visualization button

  ![dashboard-2](/images/dashboard-2.png)

13. Chose the "Bar Vertical" chart type
  
    ![dashboard-3](/images/dashboard-3.png)

14. From the menu on the left drag the field "username.keyword" and drop it to the Horizontal axis

15. From the menu on the left drag the field "ammount" and drop it to the Vertical axis then select Sum

17. It should look like the following picture:

  ![dashboard-4](/images/dashboard-4.png)

17. Click the "Save and Return" button to save the new panel.

18. Click the "Save" button to save the dashboard and give it a name.
