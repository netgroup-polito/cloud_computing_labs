<table style="width:100%">
  <tr>
    <td align="left"><a href="../1.3/README.md">⬅️ Previous</a></td>
    <td align="right"><a href="../1.5/README.md">Next ➡️</a></td>
  </tr>
</table>

# 4. Resource allocation and scaling

This section is inspired by the article “*Kubernetes best practices:
Resource requests and limits*” from the [Google GCP blog](https://cloud.google.com/blog/products/gcp/kubernetes-best-practices-resource-requests-and-limits).

So far, we analyzed how we can simply schedule pods on Kubernetes,
making the assumption that the cluster will always be able to host our
application. However, when Kubernetes schedules a Pod, it is important
that the containers have enough resources to actually run. If you
schedule a large application on a node with limited resources, it is
possible for the node to run out of resources, resulting in unexpected
behavior.

They are many reasons for an application to require more resources. This
could be caused by horizontal scaling policies, where more replicas are
spun to decrease latency, to unexpected behavior that increases the
CPU/Memory consumption.

In this section, we first understand how to take control of the cluster
avoiding bad resource allocation. Then, we focus on scaling an
application dynamically by using its resource consumption as target.

## Pre-requirements

However, before digging in resource consumption management, we install
the *metrics-server* to rely on a straightforward mechanism to scrape
information about real consumption.

### Deploying the metrics server

In order to collect information about pods and nodes resource
consumption, you need a metrics server:
```sh
    kubectl apply -f https://raw.githubusercontent.com/netgroup-polito/cloud_computing_labs/refs/heads/main/lab3/1.4/metric-server.yaml
```

The metric service is now available in Kubernetes:
```sh
    kubectl get pods -n kube-system -o wide
```
You can see a new entry for the pod, with the name
`metrics-server-XXXXXXXXX`.

Now, *describe* the metrics server deployment and check that one replica
is available with the following command:
```sh
    kubectl -n kube-system describe deployment metrics-server
```
The Horizontal Pod Autoscaling controller makes use of the metrics
provided by the `metrics.k8s.io` API, which is provided by the metrics
server. After the metrics server is running on your cluster and has had
a chance to collect metrics from the cluster (give it a minute or so),
you should be able to use the `kubectl top` command to see the resource
usage of the pods and nodes in your cluster.

By exploiting the metric server, you can see the resource usage of the
pods and nodes in your cluster. Run:
```sh
    kubectl top pod -A
```
This command prints for each pod the amount of CPU and Memory currently
used. After some time (e.g., one minute), you are able to see the usage
of nodes as well:
```sh
    kubectl top nodes
```
## Requests and Limits

When you define a pod template, it is possible to specify how many
resources each container needs. There are two main types of resources:
**CPU** and **Memory**. The Kubernetes scheduler uses them to figure out
**where** to run your pods and each Kubelet to instrument the kernel to
limit resource consumption of different pods.

As clearly stated in the Kubernetes[^5], you can specify two kind of
resource quantities: **requests** and **limits**.

When you specify the **request** for containers in a Pod:

- the **Scheduler** uses this information to decide which node to place
  the Pod on.

- the **Kubelet** also reserves at least the requested amount of that
  system resource specifically for that container to use.

When you specify a resource **limit** for a Container:

- the **Kubelet** enforces those limits so that the running container is
  not allowed to use more of that resource than the limit you set.

### CPU

Limits and requests for CPU resources are measured in CPU units. In
fact, one CPU, in Kubernetes, is equivalent to 1 vCPU/Core for cloud
providers and 1 hyperthread on bare-metal Intel/AMD processors.

It is important to understand that the CPU is always requested as an
absolute quantity, never as a relative quantity; 0.1 is the same amount
of CPU on a single-core, dual-core, or 48-core machine. Fractional
requests for CPU resources are allowed. For example, if your container
needs 3 full cores to run, you would put the value “3000m”. If your
container only needs half of a core, you would put a value of “500m”.

> [!TIP]
> Unless your app is specifically designed to take advantage of
multiple cores (scientific computing and some databases come to mind),
it is usually a best practice to keep the CPU request at ‘1’ or below,
and run more replicas to scale it out. This gives the system more
flexibility and reliability.

> [!NOTE]
> If your app starts hitting your CPU limits, the container
will be artificially restricted or **throttled**. However, it will not
be terminated or evicted. Analyzed in
<a href="#sec:livenessprobe" data-reference-type="ref+label"
data-reference="sec:livenessprobe">4.1</a>, liveness checks can be used
to be sure that your application is fully functional.

> [!NOTE]
> If you put in a value larger than the core count of your
biggest node, your pod will never be scheduled.

### Memory

Limits and requests for memory are measured in bytes. You can express
memory as a plain integer or as a fixed-point number using one of these
suffixes: E, P, T, G, M, K. You can also use the power-of-two
equivalents: Ei, Pi, Ti, Gi, Mi, Ki. For example, the following
represent roughly the same value: 128974848, 129e6, 129M, 123Mi

> [!NOTE]
> Like CPU, if you put in a memory request that is larger than
the amount of memory on your nodes, the pod will never be scheduled.

> [!WARNING]
> **Unlike CPU resources, memory cannot be compressed**. In other
words, if a container **goes past its memory limit it will be
terminated** (i.e. OOMKilled).

### Inspect available resources

To inspect the occupation of node, you can rely on:
```sh
    kubectl top nodes
```
To observe the globally available resources of a certain node, we can
use:
```sh
    kubectl get nodes worker-1 -o yaml
```
Questions:

- Which are the fields in the node resource that state the available
  resources? (They are two)

- Explain the meaning of those fields by leveraging the “kubectl
  explain“ tool.

### Example

The following presents an example pod configuration, composed of two
containers, each with the amount of resources specified in the
appropriate section:

enumerate itemize

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  [...]
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
  - name: log-aggregator
    image: my-wonderful-sidecar
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

### Exercise

First, let run a pod which attempts to consume an entire CPU:
```sh
    kubectl create deploy stress \
        --image=harbor.crownlabs.polito.it/proxy/alexeiled/stress-ng:0.12.05 \
        -- /stress-ng --cpu=1
```
- How many CPUs is the pod consuming? (It takes some time before the
  metrics are shown by the `kubectl top po` command)

Let us edit the deployment to change the maximum amount of CPU consumed
(e.g., 100m), adding the resources section as showed in the pod example
at <a href="#ref:resourceexample" data-reference-type="ref"
data-reference="ref:resourceexample">5.2.4</a>:
```sh
    kubectl edit deploy stress
```
Alternatively, you can dump the manifest to a file, modify it and then
re-apply it:
```sh
    kubectl get deploy stress -o yaml > stress-deploy.yaml
    kubectl apply -f stress-deploy.yaml
```
Check again the resource usage. How many CPUs is the pod consuming now?

Before moving to the next section, let delete the `stress` deployment:
```sh
    kubectl delete deploy stress
```
## Horizontal pod autoscaling

The Horizontal Pod Autoscaler (HPA) automatically scales the number of
pods in a replication controller, deployment, replica set or stateful
set based on observed CPU utilization. In this example, we demonstrate
Horizontal Pod Autoscaler using a custom docker image based on the
`php-apache` image.

## Autoscaling pods based on CPU usage

To test the scaling of the application, we use a php server based on the
apache web server. It defines an `index.php` page which performs some
CPU intensive computations:
```php
    <?php
        $x = 0.0001;
        for ($i = 0; $i <= 1000000; $i++) {
        $x += sqrt($x);
        }
        echo "OK!";
    ?>

First, start a deployment running the image containing the server, and
expose it as a service:
```sh
    kubectl create deploy php-apache --image=k8s.gcr.io/hpa-example --port=80
    kubectl expose deploy php-apache --port=80
```
Let’s edit the deployment, specifying the requests and limits in the
resources section:
```yaml
    ...
        resources:
          requests:
            memory: "128Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "250m"
    ...
```
Now that the server is running, we create the autoscaler using
`kubectl autoscale` command.
```sh
    kubectl autoscale deployment php-apache --cpu-percent=25 --min=1 --max=10
```
This command creates a Horizontal Pod Autoscaler (HPA) that maintains
between 1 and 10 replicas of the Pods controlled by the php-apache
deployment we created in the first step of these instructions. Roughly
speaking, the HPA will increase and decrease the number of replicas (via
the deployment) to maintain an average CPU utilization across all Pods
of 25%.

> [!NOTE]
> We set the value of 25% CPU utilization for testing purposes
and to not overload the cluster. In production-grade solutions, a
reasonable value is 50%, but this may change according to the business
logic. For more details on the autoscaling algorithm see the (official
documentation)[https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale].

Check the current status of autoscaler by running:
```sh
    kubectl get hpa
```
The output shows that the current CPU consumption is 0% as we are not
sending any requests to the server (the `CURRENT` column shows the
average across all the pods controlled by the corresponding deployment).
```
    NAME       REFERENCE                TARGET    MINPODS MAXPODS   REPLICAS   AGE
    php-apache Deployment/php-apache    0% / 25%  1       10        1          18s
```
### Increase load

To see how the autoscaler reacts to increased load, start a container,
and send an infinite loop of queries to the php-apache service (please
run it in a different terminal):
```sh
    kubectl run -it --rm load-generator \
        --image=harbor.crownlabs.polito.it/proxy/library/busybox:1.34 -- /bin/sh
```
    If you don't see a command prompt, try pressing enter.
```sh
    while true; do wget -q -O- http://php-apache.default.svc.cluster.local; done
```
Within a minute or so, stop the pod that generated the requests with
`CTRL + C, CTRL + D`. Then monitor the higher CPU load by executing:
```sh
    kubectl get hpa
```
The output shows that the current CPU consumption has increased.
```
    NAME       REFERENCE              TARGET      CURRENT  MINPODS  MAXPODS  REPLICAS
    php-apache Deployment/php-apache  305% / 25%  305%     1        10       1
```
In this snapshot, CPU consumption has increased to 305% of the request.
As a result, the deployment was resized to 7 replicas, as confirmed by
the command:
```sh
    kubectl get deployment php-apache
```
with output:
```
    NAME            READY       UP-TO-DATE      AVAILABLE       AGE
    php-apache      7/7         7               7               19m
```
It may take a few minutes to stabilize the number of replicas. Since the
amount of load is not controlled in any way it may happen that the final
number of replicas will differ from this example.

### Stop load

Before finishing the experiment, stop the user load. In the terminal
where we created the container with the `busybox` image, terminate the
load generation by typing `<Ctrl> + C`.

After a minute or so, verify the resulting state:
```sh
    kubectl get hpa
```
the output confirms the CPU utilization is 0%.
```
    NAME       REFERENCE                TARGET    MINPODS MAXPODS   REPLICAS   AGE
    php-apache Deployment/php-apache    0% / 25%  1       10        1          11m
```
To verify the number of replicas, run the following command:
```sh
    kubectl get deployment php-apache
```
since CPU utilization is dropped to 0, HPA autoscaled the number of
replicas back down to 1.
```
    NAME         READY   UP-TO-DATE   AVAILABLE   AGE
    php-apache   1/1     1            1           27m
```
Additional metrics can be used when autoscaling the `php-apache`
Deployment by making use of the `autoscaling/v2beta2` API version.
Examples of other metrics are requests per second, number of HTTP GET
requests, and so on.

**To sum up.** Kubernetes comes with many services already deployed, yet
it may need to be extended for production purposes. To do so, custom
services such as *metrics-server* can be installed and communicates with
the API service. Hence, other apps can exploit new services as well. In
this section, we installed the *metrics-server* to make the services
scale up and down according to the incoming traffic.
