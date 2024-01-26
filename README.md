# Setup Redis HA in Kubernetes

## Prerequisites
- Kubernetes, e.g. Minikube
- Helm v3

## Prepare
- add Bitnami Helm repo: `helm repo add bitnami https://charts.bitnami.com/bitnami`
- clone git repo: `git clone https://github.com/reneradoi/redis-k8s.git && cd redis-k8s`
- create service account for sentinel (this is missing in Bitnami's Helm Chart, seems to be a bug): `kubectl apply -f sa-redis-sentinel.yaml`
- install the chart into default namespace: `helm upgrade --install redis-sentinel bitnami/redis --values values.yaml`
- check the deployment:
```
rene:~/redis-k8s$ helm list
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART          APP VERSION
redis-sentinel  default         4               2024-01-26 08:44:51.2538825 +0100 CET   deployed        redis-17.9.2   7.0.10

rene:~/redis-k8s$ kubectl get sts
NAME                  READY   AGE
redis-sentinel-node   3/3     97m

rene:~/redis-k8s$ kubectl get pods
NAME                    READY   STATUS    RESTARTS   AGE
redis-client            1/1     Running   0          85m
redis-sentinel-node-0   2/2     Running   0          98m
redis-sentinel-node-1   2/2     Running   0          97m
redis-sentinel-node-2   2/2     Running   0          96m
```

## Usage
To get your password run:

`export REDIS_PASSWORD=$(kubectl get secret --namespace default redis-sentinel -o jsonpath="{.data.redis-password}" | base64 -d)`

### To connect to your Redis server

1. Run a Redis pod that you can use as a client:

`kubectl run --namespace default redis-client --restart='Never' --env REDIS_PASSWORD=$REDIS_PASSWORD --image docker.io/bitnami/redis:7.2.4-debian-11-r2 --command -- sleep infinity`

2. Connect into the pod:

`kubectl exec -it redis-client --namespace default -- bash`

3. Connect using the Redis CLI:

`REDISCLI_AUTH="$REDIS_PASSWORD" redis-cli -h redis-sentinel -p 6379 # Read only operations`
`REDISCLI_AUTH="$REDIS_PASSWORD" redis-cli -h redis-sentinel -p 26379 # Sentinel access`

### To connect to your database from outside the cluster
create k8s port forwarding for external access:

`kubectl port-forward --namespace default svc/redis-sentinel 6379:6379`

start redis cli (needs to be installed locally):

`REDISCLI_AUTH="$REDIS_PASSWORD" redis-cli -h 127.0.0.1 -p 6379`

## Test the Failover
One of Sentinel's main features is automatic failover in case the master is no longer available.

To test this, execute the following steps:
- get current master:
```
redis-sentinel:26379> SENTINEL get-master-addr-by-name mymaster
1) "redis-sentinel-node-0.redis-sentinel-headless.default.svc.cluster.local"
2) "6379"
```

- shut down the pod that is currently running the master:
```
rene:~/redis-k8s$ kubectl delete pod redis-sentinel-node-0
pod "redis-sentinel-node-0" deleted
```

- check who is master now:
```
redis-sentinel:26379> SENTINEL get-master-addr-by-name mymaster
1) "redis-sentinel-node-2.redis-sentinel-headless.default.svc.cluster.local"
2) "6379"
```

- check what is in the logs:
```
rene:~/redis-k8s$ kubectl logs redis-sentinel-node-2 sentinel
1:X 26 Jan 2024 09:48:16.213 # +config-update-from sentinel 5add24b6f14969527b26e2b3d62ebfa73e2de554 redis-sentinel-node-1.redis-sentinel-headless.default.svc.cluster.local 26379 @ mymaster redis-sentinel-node-0.redis-sentinel-headless.default.svc.cluster.local 6379
1:X 26 Jan 2024 09:48:16.213 # +switch-master mymaster redis-sentinel-node-0.redis-sentinel-headless.default.svc.cluster.local 6379 redis-sentinel-node-2.redis-sentinel-headless.default.svc.cluster.local 6379
1:X 26 Jan 2024 09:48:16.214 * +slave slave redis-sentinel-node-1.redis-sentinel-headless.default.svc.cluster.local:6379 redis-sentinel-node-1.redis-sentinel-headless.default.svc.cluster.local 6379 @ mymaster redis-sentinel-node-2.redis-sentinel-headless.default.svc.cluster.local 6379
1:X 26 Jan 2024 09:48:16.214 * +slave slave redis-sentinel-node-0.redis-sentinel-headless.default.svc.cluster.local:6379 redis-sentinel-node-0.redis-sentinel-headless.default.svc.cluster.local 6379 @ mymaster redis-sentinel-node-2.redis-sentinel-headless.default.svc.cluster.local 6379
1:X 26 Jan 2024 09:48:16.306 * Sentinel new configuration saved on disk
1:X 26 Jan 2024 09:48:26.354 * +reboot slave redis-sentinel-node-0.redis-sentinel-headless.default.svc.cluster.local:6379 redis-sentinel-node-0.redis-sentinel-headless.default.svc.cluster.local 6379 @ mymaster redis-sentinel-node-2.redis-sentinel-headless.default.svc.cluster.local 6379
```
-> Sentinel has switched the Master to node 2, failover was completed