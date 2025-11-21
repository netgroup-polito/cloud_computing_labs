# Deploying simple applications through Kubernetes

Since you have now a fully working Kubernetes cluster, we can use it to
host some simple applications.

## Listing the running pods

First, let take a look at the running pods. You can see that there are
no pods running in the *default* namespace:
```sh
kubectl get pods -o wide
```
At the beginning, this command returns no resources because you have not
deployed any applications yet. However, you will use this command
several times in the next sections to monitor the progress of your
applications.

Instead, you can find some infrastructural pods within the *kube-system*
namespace:
```sh
kubectl get pods -n kube-system -o wide
```
The *-o wide* option gives information about the pod IPs and host
location. The `-n kube-system` option, instead, specifies the namespace
for which we ask to see the resources. `kube-system` is the namespace
for objects created by the Kubernetes system. `default`, on the other
hand, is the one for objects with no namespace explicitly configured.

## Running a stateless application & service

You can run an application by creating a Kubernetes Deployment object,
and you can describe a Deployment through a YAML file. For example, this
YAML file describes a Deployment that runs the `nginxdemos/hello` Docker
image (an NGINX server that serves a simple page containing its
hostname, IP address and port as wells as the request URI and the local
time):

enumerate itemize

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # run two replicas of the following template
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: harbor.crownlabs.polito.it/proxy/nginxdemos/hello:0.3-plain-text
        ports:
        - name: http
          containerPort: 80
```

**Warning**: YAML files (e.g. `deployment-nginx.yaml`) strongly rely on
indentation for their structure. Hence, do *not* copy the previous
snippet directly from the PDF file, since it will totally scramble the
indentation. Moreover, before moving on, verify (and fix, if necessary)
the file layout using an editor of your choice (e.g.,
`nano deployment-nginx.yaml`).

Create a Deployment based on the YAML file:
```sh
kubectl apply -f deployment-nginx.yaml
```

Display information about the Deployment:
```sh
kubectl describe deployment nginx
```

The output displays the information about the label, replicas, and the
template of the pod.

List the pods created by the deployment:
```sh
kubectl get pods -l app=nginx
```

The command displays all the pods in the cluster with the specified
label, as specified by the `-l` parameter. In our case there are two
running pods, make sure the status of each is `RUNNNING`.

Try out the following command to further scale the deployment:
```sh
kubectl scale deployment nginx --replicas=5
```

This command can take a few seconds before all the 5 replicas are ready.
Monitor the status of the pods with the `kubectl get pods -l app=nginx`
command.

Try to delete one nginx pod with:
```sh
kubectl delete pod <pod_name>
```

and see what happens, by using (to be executed first in another
terminal):
```sh
kubectl get deployment nginx --watch
```

to monitor the changes. The `--watch` flag to start watching updates to
a particular object, press `CTRL + C` to stop the watch process. Scale
down the deployment to two instances as before.
```sh
kubectl scale deployment nginx --replicas=2
```

You can now see that the application consists of two pods only.

With your pods running, it is now time to put them behind a service, an
abstraction which exposes a set of pods through a single network
endpoint. Use the `kubectl create` command to create the service.
```sh
kubectl create -f service-nginx.yaml
```

The YAML file describing the service should be like the following:
enumerate itemize

``` yaml
kind: Service
apiVersion: v1
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

The IP address of the service can be accessed via:
```sh
kubectl get services nginx
```

The output reports service information, and the IP can be used to access
the service. Use first *ClusterIP* as *ServiceType* and try to access
the application from the master node. Use `curl` of `wget` to request
web pages of the service. You can use the output templating feature of
kubectl to run curl as a one-line command.
```sh
curl -ks http://$(kubectl get svc nginx -o=jsonpath="{.spec.clusterIP}")
```
**Question:** This command has to be executed from a node of the cluster
to work. Why?

Then, try to use the *NodePort* service type and try to access the app
again. When using a *NodePort* service, you can directly contact one of
the node IP addresses to access the app. Modify the type of the service
by applying the patch to the existing service.
```sh
kubectl apply -f service-nodeport.yaml
```

The patch specifies which port to expose and the new ServiceType, by
means of `nodePort`.

enumerate itemize

``` yaml
kind: Service
apiVersion: v1
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30100
  type: NodePort
```

The two services have different behaviors, indeed:

- `ClusterIP`: Exposes the Service on a cluster-internal IP. Choosing
  this value makes the service only reachable from within the cluster.
  This is the default ServiceType.

- `NodePort`: Exposes the Service on each Node’s IP at a static port
  (the NodePort). A ClusterIP Service, to which the NodePort Service
  routes, is automatically created. You are able to contact the NodePort
  Service, from outside the cluster, by requesting
  `<NodeIP>:<NodePort>`.

Other two service types are available in Kubernets, `LoadBalancer` and
`ExternalName`, but those are out of scope in this lab.

When using the `NodePort` service type, you can also connect to the
application from the web browser of your management workstation at
<a href="http://&lt;IP_MASTER_NODE&gt;:30100"
class="uri">http://&lt;IP_MASTER_NODE&gt;:30100</a>.

**Warning:** The cluster nodes can only be accessed from your other VMs
hosted by CrownLabs.

Before proceeding to the next section, it is possible to delete the
deployed objects:
```sh
kubectl delete -f service-nginx.yaml
kubectl delete -f service-nginx-nodeport.yaml
kubectl delete -f deployment-nginx.yaml
```

## Stateful applications & storage provisioner

On-disk files in a container are ephemeral, which is problematic in case
of stateful applications. First, when a container crashes, the *kubelet*
restarts it, but the files are lost — the container starts with a clean
state. Second, when running containers together in a pod, it is often
necessary to share files among them.

To execute stateful applications in Kubernetes, it is possible to
leverage the *PersistentVolume (PV)* and *PersistentVolumeClaim (PVC)*
abstractions. A PersistentVolume is a piece of storage in the cluster
that has been provisioned either by an administrator or dynamically
using Storage Classes. A PersistentVolumeClaim (PVC) is an abstract
request for storage by a user, which eventually triggers the creation of
the associated PersistentVolume (when dynamic creation is supported) and
binds a pod to it.

### Creating the Storage Class

In this lab we leverage `Local Persistent Volumes`, hence backing
PersistentVolumeClaim objects with local storage on the physical nodes.
To simplify the provisioning of the volumes, we will use the *Local Path
Provisioner* plugin[^3].

**Warning**: Local volumes are not appropriate for most applications.
Using local storage ties your application to that specific node, making
it harder to be scheduled. If that node or local volume encounters a
failure and becomes inaccessible, then that pod also becomes
inaccessible. For these reasons, in production environments, it is
recommended to use more advanced solutions, such a those provided by
tools like NFS, Ceph, and OpenStack Cinder.

Some use cases that are suitable for local storage include: *(i)*
caching of datasets that can leverage data gravity for fast processing,
*(ii)* distributed storage systems that shard or replicate data across
multiple nodes. Examples include distributed datastores like Cassandra,
or distributed file systems like Gluster or Ceph. Suitable workloads are
tolerant of node failures, data unavailability, and data loss. They
provide critical, latency-sensitive infrastructure services to the rest
of the cluster, and should run with high priority compared to other
workloads.

At this point, the first step that we need to perform concerns with the
deployment of *Local Path Provisioner*. Along with the creation of the
appropriate resources (e.g., deployment), the manifest also configures
the corresponding *StorageClass*.

```sh
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.23/deploy/local-path-storage.yaml
```

### Deploying MySQL

Once a StorageClass is available, it is possible to proceed running a
stateful application. To this end, the following listing describes a
StatefulSet that leads to the creation of one replica of a MySQL
container. The replica is associated to a persistent volume
automatically created through the defined template (i.e., specifying the
StorageClass, and the volume size), and mounted at the `/var/lib/mysql`
path within the container.

enumerate itemize

``` yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  serviceName: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: harbor.crownlabs.polito.it/proxy/library/mysql:5.6
        name: mysql
        env:
        # Use a secret in real usage
        - name: MYSQL_ROOT_PASSWORD
          value: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: mysql-persistent-storage
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
      storageClassName: local-path
```

**Note**: The mysql password is defined in the configuration, and this
is insecure. Kubernetes Secrets represent a more secure solution.

The above manifest can be applied through:
```sh
kubectl apply -f mysql-statefulset.yaml
```

You can display information about the StatefulSet through:
```sh
kubectl describe statefulset mysql
```

The output highlights the number of replicas, the volume claims and the
mount points. Make sure that the generated pod is correctly running:
```sh
kubectl get pods -l app=mysql
```

Additionally, you can inspect the PersistentVolumeClaim automatically
generated by the StatefulSet through:
```sh
kubectl describe pvc mysql-persistent-storage-mysql-0
```

Specifically, you can check that the status is *Bound*, and the capacity
is as expected.

### Accessing the DB

Once the pod is ready, it is possible to connect to the MySQL server:
```sh
kubectl exec -it statefulset/mysql -- mysql --user=root --password=password
```

You have entered the pod, and opened the mysql console. If the database
is up and running, you should see an output like this:
```sh
    mysql>
```
Once you are connected to the MySQL server, a welcome message is
displayed and the `mysql>` prompt appears, at which SQL statements are
to be sent to the server for execution. You can create a new database by
using a `CREATE DATABASE` statement:
```sh
    mysql> CREATE DATABASE pets;
    Query OK, 1 row affected (0.01 sec)
```
and check if the database has been created:
```sh
    mysql> SHOW DATABASES;
    +--------------------+
    | Database           |
    +--------------------+
    | information_schema |
    | mysql              |
    | performance_schema |
    | pets               |
    | sys                |
    +--------------------+
    5 rows in set (0.00 sec)

Exit the pod typing `CTRL + D`.
```
Now, test the persistence of the data, deleting the MySQL pod.
```sh
kubectl delete pod mysql-0
```
After a minute, list again the available databases, repeating the
previous steps:
```sh
    mysql> SHOW DATABASES;
    +--------------------+
    | Database           |
    +--------------------+
    | information_schema |
    | mysql              |
    | performance_schema |
    | pets               |
    | sys                |
    +--------------------+
    5 rows in set (0.00 sec)
```
You can see the `pets` database is still available.

**Note.** Do not change the number of replicas of MySQL: the database is
not configured to work in a high-availability setup.

Before proceeding to the next section, it is possible to delete the
deployed objects:
```sh
kubectl delete statefulset mysql
kubectl delete pvc mysql-persistent-storage-mysql-0
```