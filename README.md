# k8s first steps

Minikube

[minikube installation](https://minikube.sigs.k8s.io/docs/start/?arch=%2Flinux%2Fx86-64%2Fstable%2Fbinary+download)

```bash
minikube status
minikube dashboard # optional

# list disks
minikube ssh -- df -h /
minikube ssh "df -h"

docker ps -a # minikube is a docker container
```

k8s

```bash
kubectl get nodes
kubectl get pod
kubectl get all

# first dependencies
kubectl apply -f mongo-config.yaml
kubectl apply -f mongo-secret.yaml

# create db
kubectl apply -f mongo.yaml

# observe
kubectl get all
kubectl get configmap
kubectl get secret
kubectl describe service mongo-service
kubectl describe pod mongo-deployment-58c86bff6f-sklk4

# fix
kubectl apply -f mongo-secret.yaml
kubectl apply -f mongo.yaml
kubectl delete secret monog-secret # typographical mistake
kubectl delete pod -l app=mongo

# access
kubectl port-forward service/mongo-service 27017:27017
mongosh mongodb://mongouser:mongopassword@localhost:27017

# mongodb
show dbs
use local
show collections
db.records.insertOne({ name: "test", value: 123 })
db.records.find()
# delete pod and check if record exists
kubectl delete pod -l app=mongo
...
# rollback if you break smt
kubectl rollout undo

# list disks
kubectl get pvc

# scale up for testing
kubectl scale deployment mongo-deployment --replicas=3
```

The new 2 pods will crash since the disk is used by the first.

```bash
# attach to a pod
kubectl exec -it pod/mongo-deployment-54d8f66f54-bz9gr -- bash
# tail the logs
kubectl logs -f mongo-deployment-54d8f66f54-h6hlh
# clean up
kubectl scale deployment mongo-deployment --replicas=1
kubectl delete pod mongo-deployment-54d8f66f54-h6hlh mongo-deployment-54d8f66f54-rjkl5

kubectl delete replicaset mongo-deployment-58c86bff6f mongo-deployment-d4cccb59f
```

To make change from Deployment to StatefulSet:

```bash
kubectl delete deployment mongo-deployment
kubectl delete pvc mongo-pvc
kubectl apply -f mongo.yaml
```

Changes in disks.

```bash
kubectl get pvc

NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
mongo-pvc   Bound    pvc-886f82d5-765f-4eee-8385-7b2930512db7   1Gi        RWO            standard       <unset>                 20m

kubectl get pvc

NAME                                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
mongo-storage-mongo-statefulset-0   Bound    pvc-9bc46e3b-aefa-42a8-8b95-6ceb850adc88   1Gi        RWO            standard       <unset>                 2m57s
```

Scaled application.

```bash
kubectl get pvc

NAME                                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
mongo-storage-mongo-statefulset-0   Bound    pvc-9bc46e3b-aefa-42a8-8b95-6ceb850adc88   1Gi        RWO            standard       <unset>                 8m45s
mongo-storage-mongo-statefulset-1   Bound    pvc-624ba171-6244-442a-9151-97affcfd81a5   1Gi        RWO            standard       <unset>                 119s
mongo-storage-mongo-statefulset-2   Bound    pvc-297e545e-b43c-4c3b-a545-d0ecb539043d   1Gi        RWO            standard       <unset>                 118s
```

Test the replication.

```bash
kubectl scale statefulset mongo-statefulset --replicas=3

kubectl exec -it mongo-statefulset-0 -- mongosh -u mongouser -p mongopassword --eval "db.local.insert({name: 'Insert Test'})"
kubectl delete pod mongo-statefulset-0
kubectl exec -it mongo-statefulset-0 -- mongosh -u mongouser -p mongopassword --eval "db.local.find()"

kubectl exec -it mongo-statefulset-0 -- mongosh -u mongouser -p mongopassword --eval 'rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongo-statefulset-0.mongo-internal:27017" },
    { _id: 1, host: "mongo-statefulset-1.mongo-internal:27017" },
    { _id: 2, host: "mongo-statefulset-2.mongo-internal:27017" }
  ]
})'
```

Add headless serves to sync disks.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongo-internal # headless service for internal communication between MongoDB pods
spec:
  clusterIP: None # headless service
  selector:
    app: mongo
  ports:
    - port: 27017
      targetPort: 27017
```

Apply the changes.

```bash
kubectl delete statefulset mongo-statefulset
kubectl apply -f mongo.yaml

kubectl exec -it mongo-statefulset-0 -- mongosh -u mongouser -p mongopassword --eval 'rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongo-statefulset-0.mongo-internal.default.svc.cluster.local:27017" },
    { _id: 1, host: "mongo-statefulset-1.mongo-internal.default.svc.cluster.local:27017" },
    { _id: 2, host: "mongo-statefulset-2.mongo-internal.default.svc.cluster.local:27017" }
  ]
})'

kubectl exec -it mongo-statefulset-0 -- mongosh -u mongouser -p mongopassword --eval "db.local.insert({name: 'Insert Test'})"
kubectl exec -it mongo-statefulset-0 -- mongosh -u mongouser -p mongopassword --eval "db.local.find()"
kubectl exec -it mongo-statefulset-1 -- mongosh -u mongouser -p mongopassword --eval "db.local.find()"
...
```

Get the manifest and cluster information.

```bash
kubectl get statefulset mongo-statefulset -o yaml

kubectl config current-context
kubectl config view --minify
kubectl config get-clusters
kubectl get ns # reads `~/.kube/config` or KUBECONFIG
...
kubectl get pod -n {name}
kubectl get ingress -n {name}
kubectl get svc -n {name}
```

If you delete the project files you can recover them from etcd.

```bash
kubectl get secret mongo-secret -o yaml > recovered-secret.yaml

# unrecoverable
kubectl delete -f mongo.yaml

# then only github can recover it
git pull
kubectl apply -f .
```
