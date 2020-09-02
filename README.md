# Deploying StorageGRID on Kubernetes
Sample configuration files for Kubernetes deployments

## Read the full article on NetApp's thePub:
[Deploying StorageGRID in a Kubernetes Cluster](https://netapp.io/2019/01/15/deploying-storagegrid-in-a-kubernetes-cluster/)

### Create a “storagegrid” namespace
File: [0-storagegrid-namespace.yaml](11.4.0/0-storagegrid-namespace.yaml)
```	
$ kubectl create -f 0-storagegrid-namespace.yaml
namespace "storagegrid" created
```
### Create standard-xfs StorageClass (Example based on GKE)
File: [1-gcp-gke-storageclass-sg-xfs.yaml](11.4.0/1-gcp-gke-storageclass-sg-xfs.yaml)
```
$ kubectl create -f 1-gcp-gke-storageclass-sg-xfs.yaml
storageclass.storage.k8s.io/sg-xfs created

$ kubectl get storageclass
NAME                 PROVISIONER            AGE
standard (default)   kubernetes.io/gce-pd   17m
sg-xfs               kubernetes.io/gce-pd   4m32s
```
### Deploy the primary Admin Node
File: [2-primary-admin.yaml](11.4.0/2-primary-admin.yaml)
```
$ kubectl create --namespace storagegrid -f 2-primary-admin.yaml
persistentvolumeclaim/var-local-dc1-adm-0 created
persistentvolumeclaim/mysql-dc1-adm-0 created
persistentvolumeclaim/audit-dc1-adm-0 created
service/pod-dns created
service/dc1-admin-node created
statefulset.apps/dc1-adm created
```
### Verify the StatefullSet deployed and the pod started.
```
$ kubectl get statefulsets dc1-adm --namespace storagegrid
NAME      READY   AGE
dc1-adm   0/1     49s

$ kubectl get pod dc1-adm-0 --namespace storagegrid
NAME        READY   STATUS              RESTARTS   AGE
dc1-adm-0   0/1     ContainerCreating   0          100s

$ kubectl get pod dc1-adm-0 --namespace storagegrid
NAME        READY   STATUS    RESTARTS   AGE
dc1-adm-0   1/1     Running   0          2m2s
```
### Verify the Admin Node is waiting for configuration (last line in this output).

Note: This command only shows the last three lines.
```
$ kubectl logs dc1-adm-0 --tail 3 --namespace storagegrid
[INSG] Grid Manager has been started.
[INSG] Please direct your browser to http://10.32.1.3
[INSG] Waiting for configuration information
```
### Deploy the Storage Nodes
File: [3-storage.yaml](11.4.0/3-storage.yaml)
```
$ kubectl create -f 3-storage.yaml --namespace storagegrid
persistentvolumeclaim/var-local-dc1-sn-0 created
persistentvolumeclaim/rangedb0-dc1-sn-0 created
persistentvolumeclaim/rangedb1-dc1-sn-0 created
persistentvolumeclaim/rangedb2-dc1-sn-0 created
persistentvolumeclaim/var-local-dc1-sn-1 created
persistentvolumeclaim/rangedb0-dc1-sn-1 created
persistentvolumeclaim/rangedb1-dc1-sn-1 created
persistentvolumeclaim/rangedb2-dc1-sn-1 created
persistentvolumeclaim/var-local-dc1-sn-2 created
persistentvolumeclaim/rangedb0-dc1-sn-2 created
persistentvolumeclaim/rangedb1-dc1-sn-2 created
persistentvolumeclaim/rangedb2-dc1-sn-2 created
service/dc1-storage-node created
statefulset.apps/dc1-sn created
```
### Verify the StatefulSet deployed and all the pods started.
```
$ kubectl get statefulsets dc1-sn --namespace storagegrid
NAME     READY   AGE
dc1-sn   0/3     55s
 
$ kubectl get pod dc1-sn-0 dc1-sn-1 dc1-sn-2 --namespace storagegrid
NAME       READY   STATUS              RESTARTS   AGE
dc1-sn-0   0/1     ContainerCreating   0          90s
dc1-sn-1   0/1     ContainerCreating   0          90s
dc1-sn-2   0/1     ContainerCreating   0          90s

$ kubectl get statefulsets dc1-sn --namespace storagegrid
NAME     READY   AGE
dc1-sn   3/3     2m46s

$ kubectl get pod dc1-sn-0 dc1-sn-1 dc1-sn-2 --namespace storagegrid
NAME       READY   STATUS    RESTARTS   AGE
dc1-sn-0   1/1     Running   0          2m48s
dc1-sn-1   1/1     Running   0          2m48s
dc1-sn-2   1/1     Running   0          2m48s
```
### Verify that each Storage Node is waiting for approval (last line in each output).

Note: These commands only shows the last three lines.
```
$ kubectl logs dc1-sn-0 --tail 3 --namespace storagegrid
[INSG]
[INSG]
[INSG] Please approve this node on the Admin Node GMI to proceed...

$ kubectl logs dc1-sn-1 --tail 3 --namespace storagegrid
[INSG]
[INSG]
[INSG] Please approve this node on the Admin Node GMI to proceed...

$ kubectl logs dc1-sn-2 --tail 3 --namespace storagegrid
[INSG]
[INSG]
[INSG] Please approve this node on the Admin Node GMI to proceed...

```
### Verify the services started.
```
$ kubectl get service --namespace storagegrid
NAME                   TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)                                   AGE
dc1-admin-node         LoadBalancer   10.0.21.175   34.68.210.219    80:30251/TCP,443:32511/TCP,22:30667/TCP   18m
dc1-storage-node       LoadBalancer   10.0.23.55    146.148.74.155   18082:30780/TCP                           13m
pod-dns                ClusterIP      None          <none>           12345/TCP                                 18m
```
### Direct your browser to https://\<EXTERNAL-IP\> (of the dc1-admin-node service)
Perform a standard StorageGRID install
