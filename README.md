# Distributed Application Runtime
![](/data/home-title.png)

In one simple sentence, 'DAPR simplifies developer life!'

Make DAPR your friend and say goodbye to the hassle of using different libraries for different resources - it's the one-stop solution for a streamlined development experience!

Following are some of the attributes of dapr!  
**Consistency**  
**Speed**  

Applies to  
**Brown field**  (or)
**Green field**  


## Outcome

The example demonstrates how DAPR provides an abstraction layer, **State Management**, for the backend state store, allowing applications to be written without relying on Redis libraries.    

## Method

We will be using a simple pod and exec-into (login) pod and use curl to issue POST or GET calls. 


## DAPR install

You need to be an admin (for production like install, refer https://docs.dapr.io/operations/hosting/kubernetes/kubernetes-production/ )
Verify context and install DAPR 

```
kubectl config get-contexts

dapr init -k

k get all -n dapr-system
```
Verify components in running state  


## Install Redis in our cluster
Install redis, display password, set environment variable with password, and store password in K8S secret store
```
helm install myredis bitnami/redis
echo $(kubectl get secret --namespace default myredis -o jsonpath="{.data.redis-password}" | base64 -d)  
export REDIS_PASSWORD=$(kubectl get secret --namespace default myredis -o jsonpath="{.data.redis-password}" | base64 -d)  
k create secret generic redis --from-literal=redis-password=$(kubectl get secret --namespace default myredis -o jsonpath="{.data.redis-password}" |
base64 -d)  
```

## Use Redis as the backend for our **Component** named *statestore* 
*statestore* is the name of the component that we will be using in our POST or GET calls in app.  

Apply this following yaml for creating Component with state type
```
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.redis
  version: v1
  metadata:
  - name: redisHost
    value: myredis-master.default.svc.cluster.local:6379
  - name: redisPassword
    secretKeyRef:
      name: redis
      key: redis-password
auth:
  secretStore: kubernetes
```

At this time, we have deployed Redis locally in the cluster, configured Dapr Component to communciate with Redis with the secret stored in K8S secret.   


`dapr components -k`

## Use our app to write data into Redis (indirectly via Dapr)
Lets write some data into Redis from our app pod via Dapr
```
k exec app1 -it -- sh
curl -X POST -H "Content-Type: application/json" -d '[{ "key": "srinman", "value": "hello srinman!"}]' http://localhost:3500/v1.0/state/statestore
curl -X POST -H "Content-Type: application/json" -d '[{ "key": "bob", "value": "hello bob!"}]' http://localhost:3500/v1.0/state/statestore
```


## Validation of data in Redis
In order to check the content of Redis, we could use Redis client. Following command deploy Redis client as a pod 
```
kubectl run --namespace default redis-client --restart='Never'  --env REDIS_PASSWORD=$REDIS_PASSWORD  --image docker.io/bitnami/redis:7.0.8-debian-11-r0 --command -- sleep infinity
```

Use following commands to exec into client, and connect to redis master.  Use keys * command to list all keys stored. 
```
kubectl exec --tty -i redis-client --namespace default -- bash  
REDISCLI_AUTH="$REDIS_PASSWORD" redis-cli -h myredis-master  
keys *
```

## Read data

Yon can also try to read from Redis via Dapr 
```
k exec app1 -it -- sh
curl http://localhost:3500/v1.0/state/statestore/srinman
```







