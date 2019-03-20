# Deploying StorageGRID on Kubernetes
Sample configuration files for Kubernetes deployments

## Read the full article on NetApp's thePub:
[Deploying StorageGRID in a Kubernetes Cluster](https://netapp.io/2019/01/15/deploying-storagegrid-in-a-kubernetes-cluster/)

### Create a “storagegrid” namespace
File: [storagegrid-namespace.yaml](11.2.0/storagegrid-namespace.yaml)
```	
$ kubectl create -f storagegrid-namespace.yaml
namespace "storagegrid" created
```
### Create Persistent Volume Claims
This example uses Trident; however, any Kubernetes storage class can be used.

Note: Within this file, you must modify storageClassName to suit your Kubernetes system.

File: [trident-persistent-volume-claims.yaml](11.2.0/trident-persistent-volume-claims.yaml)
```
$ kubectl create --namespace storagegrid -f trident-persistent-volume-claims.yaml
persistentvolumeclaim  "var-local-dc1-adm-0"  created
persistentvolumeclaim  "var-local-dc1-sn-0"  created
persistentvolumeclaim  "var-local-dc1-sn-1"  created
persistentvolumeclaim  "var-local-dc1-sn-2"  created
persistentvolumeclaim  "mysql-dc1-adm-0"  created
persistentvolumeclaim  "audit-dc1-adm-0"  created
persistentvolumeclaim  "rangedb0-dc1-sn-0"  created
persistentvolumeclaim  "rangedb1-dc1-sn-0"  created
persistentvolumeclaim  "rangedb2-dc1-sn-0"  created
persistentvolumeclaim  "rangedb0-dc1-sn-1"  created
persistentvolumeclaim  "rangedb1-dc1-sn-1"  created
persistentvolumeclaim  "rangedb2-dc1-sn-1"  created
persistentvolumeclaim  "rangedb0-dc1-sn-2"  created
persistentvolumeclaim  "rangedb1-dc1-sn-2"  created
persistentvolumeclaim  "rangedb2-dc1-sn-2"  created
```
### Wait until all the PVCs are bound before continuing.
```
$ kubectl --namespace storagegrid get pvc
NAME  STATUS  VOLUME  CAPACITY
audit-dc1-adm-0  Bound  storagegrid-audit-dc1-adm-0-ad3f7  200Gi
mysql-dc1-adm-0  Bound  storagegrid-mysql-dc1-adm-0-ad364  200Gi
rangedb0-dc1-sn-0  Bound  storagegrid-rangedb0-dc1-sn-0-ad4a0  4Ti
rangedb0-dc1-sn-1  Bound  storagegrid-rangedb0-dc1-sn-1-ad67f  4Ti
rangedb0-dc1-sn-2  Bound  storagegrid-rangedb0-dc1-sn-2-ad85f  4Ti
rangedb1-dc1-sn-0  Bound  storagegrid-rangedb1-dc1-sn-0-ad53c  4Ti
rangedb1-dc1-sn-1  Bound  storagegrid-rangedb1-dc1-sn-1-ad720  4Ti
rangedb1-dc1-sn-2  Bound  storagegrid-rangedb1-dc1-sn-2-ad901  4Ti
rangedb2-dc1-sn-0  Bound  storagegrid-rangedb2-dc1-sn-0-ad5d1  4Ti
rangedb2-dc1-sn-1  Bound  storagegrid-rangedb2-dc1-sn-1-ad7ba  4Ti
rangedb2-dc1-sn-2  Bound  storagegrid-rangedb2-dc1-sn-2-ad98c  4Ti
var-local-dc1-adm-0  Bound  storagegrid-var-local-dc1-adm-0-ad0b5  100Gi
var-local-dc1-sn-0  Bound  storagegrid-var-local-dc1-sn-0-ad169  100Gi
var-local-dc1-sn-1  Bound  storagegrid-var-local-dc1-sn-1-ad207  100Gi
var-local-dc1-sn-2  Bound  storagegrid-var-local-dc1-sn-2-ad2b6  100Gi
```
### Create the headless DNS service

The headless DNS service will allow the use of a DNS name to reference the Admin Node.

File: [pod-dns.yaml](11.2.0/pod-dns.yaml)
```
$ kubectl create --namespace storagegrid -f pod-dns.yaml
service "pod-dns" created
```
### Deploy the Admin Node
File: [admin-node.yaml](11.2.0/admin-node.yaml)
```
$ kubectl create --namespace storagegrid -f admin-node.yaml
statefulset "dc1-adm" created
```
### Verify the StatefullSet deployed and the pod started.
```
$ kubectl get --namespace storagegrid statefulsets dc1-adm
NAME  KIND
dc1-adm  StatefulSet.v1.apps

$ kubectl get --namespace storagegrid pod dc1-adm-0
NAME  READY  STATUS  RESTARTS  AGE
dc1-adm-0  1/1  Running  0  3m
```
### Verify the Admin Node is waiting for configuration (last line in this output).

Note: This command only shows the last three lines.
```
$ kubectl logs --namespace storagegrid dc1-adm-0 -tail 3
[INSG] Grid Manager has been started.
[INSG] Please direct your browser to http://10.112.2.100
[INSG] Waiting for configuration information
```
### Deploy the Storage Nodes
File: [storage-node.yaml](11.2.0/storage-node.yaml)
```
$ kubectl create --namespace storagegrid -f storage-node.yaml
statefulset "dc1-sn" created
```
### Verify the StatefulSet deployed and all the pods started.
```
$ kubectl get --namespace storagegrid statefulsets dc1-sn
NAME  KIND
dc1-sn  StatefulSet.v1.apps
 
$ kubectl get --namespace storagegrid pod dc1-sn-0 dc1-sn-1 dc1-sn-2
NAME  READY  STATUS  RESTARTS  AGE
dc1-sn-0  1/1  Running  0  2m
dc1-sn-1  1/1  Running  0  2m
dc1-sn-2  1/1  Running  0  2m
```
### Verify that each Storage Node is waiting for approval (last line in this output – only dc1-sn-0 is shown).

Note: This command only shows the last three lines.
```
$ kubectl logs --namespace storagegrid dc1-sn-0 -tail 3
[INSG]
[INSG]
[INSG] Please approve this node on the Admin Node GMI to proceed...
```
### Deploy the Admin Node and Storage Nodes Services

* The Admin Node service will map GMI HTTPS (port 443) from the Admin Node to make it available at port 30443 of each Kubernetes node.
* The Storage Node service will map S3 protocol (port 18082) from the Storage Nodes to make them available at port 32182 of each Kubernetes node (Kubernetes will manage load-balancing).

File: [storagegrid-services.yaml](11.2.0/storagegrid-services.yaml)
```
$ kubectl create --namespace storagegrid -f storagegrid-services.yaml
service "admin-node-servcie" created
service "storage-service" created
```
Optionally if your K8s service provider has load balancing:
File: [storagegrid-loadbalanced-services.yaml](11.2.0/storagegrid-loadbalanced-services.yaml)

### Verify the services started.
```
$ kubectl get --namespace storagegrid service admin-node-servcie storage-service
NAME  CLUSTER-IP  EXTERNAL-IP  PORT(S)  AGE
admin-node-servcie  10.107.189.111  <nodes>  443:30443/TCP  1m
storage-service  10.99.231.197  <nodes>  18082:32182/TCP  1m
```
### Direct your browser to https://\<EXTERNAL-IP\>:30443/
Perform a standard StorageGRID install
