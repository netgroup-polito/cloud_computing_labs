<table style="width:100%">
  <tr>
    <td align="left"><a href="../1.2/README.md">⬅️ Previous</a></td>
    <td align="right"><a href="../1.4/README.md">Next ➡️</a></td>
  </tr>
</table>

# 3. Self-healing applications with Kubernetes

One of the most in-demand container orchestration features is monitoring
of containers’ health and ability of the orchestrator to deal with
unhealthy containers according to the specified configuration. For
example, the orchestrator can launch a new container as a replacement
for a container that is not healthy. In Kubernetes, the same technique
applies to a pod, the smallest deployable unit, consisting of one or
more containers that share the same IP address and port space. When
something goes wrong, meaning the desired state and the actual state
doesn’t match, Kubernetes will try to close the gap, forever.

The main point is, how can we determine which container is healthy and
which is not? As we know, there is a single main process that is running
in a container. Such a process can start other child processes within a
container, if necessary. Every such process, including the main process,
can have its own lifecycle - but if the main process stops, the
container stops as well.

A container is healthy, by the most general definition, if its main
process is running. If the container’s main process is terminated
unexpectedly, then the container is considered unhealthy.

> [!NOTE]
> You should monitor health only for containers that are running
permanently, such as web servers or databases. If you are running a
container that is expected to stop sometime, there is no reason to
monitor its health. Instead, you should analyze its result, such as its
exit code. You can use a Kubernetes *Job* for pods that are expected to
terminate on their own.

In addition, you should take into consideration that the container keeps
running and is considered healthy even if one or more child processes
are terminated unexpectedly. Furthermore, the main process might be
running but not working as expected, because, for example, it was not
configured properly. For such cases, we need an application-specific way
to determine the container’s health.

Each Kubernetes pod has a `phase` field, which provides a simple,
high-level summary of where the pod is in its lifecycle. The pod can be
in one of the following phases:

- `Pending` - If the pod is created, but one or more of containers are
  not running yet. For example, it can take some time for the container
  image to be downloaded over the network.

- `Running` – All the containers in the pod are running.

- `Succeeded` - All the containers in the pod are terminated with zero
  exit code.

- `Failed` - All the containers in the pod are terminated and at least
  one container failed (either exited with non-zero exit code or was
  terminated by the system).

- `Unknown` - The state of the pod could not be obtained for some
  reason.

For each pod, Kubernetes can periodically execute *liveness* and
*readiness* probes, if they are defined for the pod. A probe is an
executable action, that can check if the specified condition is met.
There are three types of actions:

- `ExecAction` - Executes a specified command in the container. The
  condition is considered successful if the command exits with zero as
  its exit code.

- `TCPSocketAction` - Performs a TCP check against the container’s IP
  address on a specified port. The condition is considered successful if
  the port is open.

- `HTTPGetAction` - Performs an HTTP GET request against the container’s
  IP address on a specified port and path. The condition is considered
  successful if the response has an HTTP status code greater than or
  equal to 200 and less than 400.

A *liveness probe* determines whether the container is running or not.
If the liveness probe fails, then Kubernetes kills the container. A new
container can be started instead, if a *restart policy* says so.
Although there is no default liveness probe for a container, you do not
necessarily need one, because Kubernetes will automatically perform the
correct action in accordance with the pod’s restart policy.

A *readiness probe* determines whether the container is ready to service
requests. If the readiness probe fails, the endpoints controller removes
the pod’s IP address from the endpoints of all services for that pod.
There is no default readiness probe for a container.

A pod has a *restartPolicy* field with possible values *Always,
OnFailure,* and *Never*. The default value is Always. The restart policy
applies to all the containers in the pod. Failed containers are
restarted on the same node with a delay that grows exponentially up to 5
minutes.

## Define and create a liveness probe for a pod

To simulate a failure in a reproducible manner, we force failures in a
Kubernetes deployment, to verify the self-healing properties of
Kubernetes. First, define a new correct pod in the `echoserver-pod.yaml`
file. In this case, we use an existing image echoserver, which is a
simple HTTP server that responds with the HTTP headers it receives:

``` yaml
apiVersion: v1 
kind: Pod 
metadata:  
  name: echoserver 
spec:  
  containers:    
    - image: gcr.io/google_containers/echoserver:1.4      
      name: echoserver      
      ports:        
        - containerPort: 8080      
      livenessProbe:        
        httpGet:          
          path: /          
          port: 8080        
        initialDelaySeconds: 15        
        timeoutSeconds: 1
```

In this example, the liveness probe is defined for the port 8080 and
root path. It specifies also other fields:

- `initialDelaySeconds` - The number of seconds after the container has
  started before liveness probes are initiated.

- `timeoutSeconds` - The number of seconds after which the probe times
  out. The default value is 1 second; the minimum value is 1.

- `periodSeconds` - How often (in seconds) to perform the probe. The
  default value is 10 seconds; the minimum value is 1.

Create the echoserver pod:
```sh
kubectl create -f echoserver-pod.yaml
```
Check that the pod is running and there are no restarts:
```sh
kubectl get pod echoserver
```
where the output is supposed to be like:
```
NAME         READY     STATUS    RESTARTS   AGE
echoserver   1/1       Running   0          15s
```
## Define and create a failing liveness probe for a pod

Edit the file echoserver-pod.yaml. Change the pod’s name to
echoserver-failing, and change the port number (`8080`) in the liveness
probe to another value, for example, `8081`:


``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: echoserver-failing
spec:
  containers:
    - image: gcr.io/google_containers/echoserver:1.4
      name: echoserver
      ports:
        - containerPort: 8080
      livenessProbe:
        httpGet:
          path: /
          port: 8081
        initialDelaySeconds: 15
        timeoutSeconds: 1
```

Since the echoserver pod will not respond on port 8081, the liveness
probe will fail. To check this behavior, create a new echoserver-failing
pod:
```sh
kubectl create -f echoserver-pod-failing.yaml
```
Then, monitor the status of the pods:
```sh
kubectl get pod --watch
```
You should notice that the restart count of the pod with the failing
liveness probe keeps increasing.
```
NAME                READY     STATUS    RESTARTS   AGE
echoserver-failing  1/1       Running   2          1m
```
As you can see, Kubernetes has restarted our pod several times. You can
run the following command to investigate what it is happening in detail:
```sh
kubectl describe pod echoserver-failing
```
In the output of `kubectl describe`, in the Events section, you can see
the following information:
```
Killing    76s  Container echoserver failed liveness probe, will be restarted
Unhealthy  56s  Liveness probe failed: ...
```
As the failure is auto-generated, it will never reach a stable state,
that satisfies the desired property. On the contrary, generally faults
are accidental, and Kubernetes guarantees the state will be eventually
accomplished, unless configuration errors are present.

Before proceeding with the next task, clean the Kubernetes environment
with:
```sh
kubectl delete pod echoserver-failing
kubectl delete pod echoserver
```
**To sum up.** Kubernetes provides key mechanisms that allow users to
monitor containers’ health and restart them in case of failures: probes
and the restart policy. Probes are executable actions, which check if
the specified conditions are met. The pod’s restart policy specifies
actions for failed containers. It is worth noting that files in a
container are ephemeral, so when a container gets restarted, the changes
to the data will be lost. If a container is not stateless, it is
necessary to use volumes for persistent storage.