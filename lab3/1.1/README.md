<table style="width:100%">
  <tr>
    <td align="left"><a href="../README.md">⬅️ Previous</a></td>
    <td align="right"><a href="../1.2/README.md">Next ➡️</a></td>
  </tr>
</table>

# 1. Installing and configuring a Kubernetes cluster with kubeadm

We start creating a new Kubernetes cluster, composed of a single control
plane node and two worker nodes
(<a href="#fig:k8s_cluster" data-reference-type="ref+label"
data-reference="fig:k8s_cluster">Figure 2.1</a>). The installation process
requires the *kubeadm*, *kubectl* and *kubelet* packages to be installed
on all servers: the CrownLabs images already include them.

> [!NOTE]
> The setup proposed for this lab does not lead to the creation
> of a *highly available cluster* (i.e., which can tolerate the failure of
> a control plane node), and it is therefore not suggested for production
> workloads. See the [official documentation](http://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/) for additional information
> about this topic. In the current setup, a failure of the control-plane
> node may lead to data loss in your cluster and you may need to re-create
> everything from scratch.

<p id="fig:k8s_cluster" align="center">
    <img src="./images/system.png" width="80%"/>

</p>
    <p align="center">
        <em>Figure 2.1 Logical setup of the Kubernetes cluster in CrownLabs.</em>
    </p>

The Kubernetes *control plane*, including as the *api server* and the
*controller manager*, governs the cluster behavior. Specifically, the
control plane maintains a record of all the Kubernetes objects in the
system, and runs continuous control loops to manage those objects’
state. At any given time, the control plane’s control loops respond to
changes in the cluster and work to make the actual state of all the
objects in the system match the desired state that you provided. When
you interact with Kubernetes, such as using the `kubectl` command-line
interface, you are communicating with your cluster’s Kubernetes control
plane.

For example, when a *deployment* is created through the Kubernetes APIs,
it specifies a new desired state for the system. The Kubernetes control
plane reacts to that object creation, and performs the operations
required to schedule the desired applications to cluster nodes, thus
making the cluster’s actual state match the desired state.

The *worker* nodes, instead, are the machines (VMs, physical servers,
etc) that run the actual applications and workflows. The Kubernetes
control plane controls each node: you will rarely interact with the
worker nodes directly.


## Configuring the control plane node

First, select a VM that is supposed to act as **control plane node** and
change the name of the machine to clarify its function in the Kubernetes
cluster:
```sh
sudo hostnamectl set-hostname control-plane-1
```
> [!WARNING]
> You have to disconnect from the VM and open another SSH
session to update the prompt.

Then, from the same node, it is possible to leverage *kubeadm* to
initialize the control-plane:
```sh
sudo kubeadm init --pod-network-cidr=10.254.0.0/16 --service-cidr=10.255.0.0/16
```
`kubeadm init` first runs a series of *pre-flight* checks to ensure that
the machine is ready to run Kubernetes, then downloads and installs the
cluster control plane components. This may take several minutes.

> [!NOTE]
> As part of the installation process, the previous command will
> print on screen a string that looks like the following:
> ```sh
> sudo kubeadm join 172.16.183.32:6443 --token lquijh.m7g8o434w21xjaew \
>     --discovery-token-ca-cert-hash sha256:2a63522f184ed9c7ab3f...
> ```
> Take note of the above string, because it will be required later in the
> installation process of the workers. In case you miss this step, do not
> worry; you can create another token later.

Once the configuration completed, it is possible to make *kubectl* work
for the current user, through the following:
```sh
mkdir -p $HOME/.kube
# The 'kubectl' command expect the KUBECONFIG to be in folder '~/.kube'
# Copy the KUBECONFIG created during the installation process in the above folder
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
> [!NOTE]
> The *kubeconfig* file contains all the parameters required
> to connect to a Kubernetes cluster (e.g., IP address of the API server, etc). 
> This data is used by `kubectl` to connect to the above cluster.

> [!NOTE]
> You can connect to multiple clusters from the same
> workstation by simply replacing/updating the *kubeconfig* file.

Kubernetes does not embed the logic to interconnect different pods.
Instead, it leverages a modular architecture based on the *Container
Network interface (CNI)* specifications. Hence, it is possible to choose
among many alternative network plugins. In this lab, we select
[Cilium](https://cilium.io/), an open source solution which provides highly scalable
networking and network policy capabilities. **Warning:** Neither Calico
nor Flannel (two very popular alternatives) do not work properly on
CrownLabs, due to the underlying technology.

Cilium can be installed through the provided CLI:
```sh
# Download the Cilium CLI
curl -L --remote-name-all https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64tar.gz
# Uncompress the TAR in the /usr/local/bin folder
sudo tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin
rm cilium-linux-amd64.tar.gz

# Install Cilium
cilium install --set ipam.mode=kubernetes
```
Once the Cilium configuration completed, the control plane node should
turn into the *Ready* status. You can check the nodes’ status through
`kubectl get nodes` command.

## Configuring worker nodes

When the control plane node is ready, access the other two VMs to join
them to the cluster as worker nodes. First, change their hostname for
easier identification through the following command (remember to close
and reopen the SSH session to update the prompt):
```sh
sudo hostnamectl set-hostname worker-<num>
```
replacing `<num>` with 1 and 2 for the two VMs.

Then, run the `kubeadm join` command echoed during the control plane
setup process (if lost, a new one can be generated through
`kubeadm token create --print-join-command` from the control plane node)
to join the worker node to the control plane. The command should be
**similar** to the following (remember to prepend `sudo`):
```sh
sudo kubeadm join 172.16.183.32:6443 --token lquijh.m7g8o434w21xjaew \
    --discovery-token-ca-cert-hash sha256:2a63522f184ed9c7ab3f...
```
The token is used for mutual authentication between the control-plane
node and the joining nodes. The token included here is secret. Keep it
safe, because anyone with this token can add authenticated nodes to your
cluster.

Repeat the join process on both worker nodes. Then, move to the control
plane and check all nodes are correctly ready through:
```sh
kubectl get nodes
```
**Question** (left to the student): why the command `kubectl get nodes`
does not work if issued on a worker node?

## Accessing the cluster from a management workstation

Instead of issuing the `kubectl` commands directly from the control
plane node, it is possible to use a separate management workstation
(e.g., a client VM in CrownLabs). To this end, it is necessary to copy
the *kubeconfig* to the workstation by typing, on the client
workstation, the following commands (remember to replace `__MASTER_IP__`
with the IP address of the control node):
```sh
mkdir -p $HOME/.kube
scp crownlabs@__MASTER_IP__:~/.kube/config $HOME/.kube/
```
> [!WARNING]
> The control plane IP (TCP port 6443) needs to be reachable
> from the management workstation for the `kubectl` commands to work.

Then you can verify `kubectl` correctly works:
```sh
kubectl get nodes
```
> [!TIP]
> To simplify the interaction with the server, it is strongly
> suggested to enable bash auto-completion (and the `k` alias) for
> `kubectl` commands:
> ```sh
> echo 'source <(kubectl completion bash)' >> $HOME/.bashrc
> echo 'alias k=kubectl' >> $HOME/.bashrc
> echo 'complete -F __start_kubectl k' >> $HOME/.bashrc
> source $HOME/.bashrc
> ```

## Deploying the Kubernetes Dashboard

To simplify the interaction with the cluster, it is possible to use the
official Kubernetes dashboard. The Dashboard UI is not deployed by
default. To deploy it, run the following command:
```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```
To protect your cluster data, the dashboard deploys with a minimal RBAC
configuration by default. Currently, it only supports Bearer Token
authentication. To create a token with administrative privileges for
this lab, you can proceed as follows:
```sh
kubectl create serviceaccount -n kubernetes-dashboard dashboard-admin
kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin \
    --serviceaccount=kubernetes-dashboard:dashboard-admin
```
Then you can retrieve the authentication token:
```sh
NAME=$(kubectl get serviceaccount -n kubernetes-dashboard dashboard-admin \
    --no-headers -o custom-columns=':.secrets[0].name')
TOKEN=$(kubectl get secret -n kubernetes-dashboard $NAME \
    -o custom-columns=":.data.token" --no-headers | base64 -d)
echo Token: $TOKEN
```
By default, the Dashboard is exposed through a ClusterIP service, which
can only be accessed from within the cluster. One of the possibilities
to access it from the management workstation is to leverage the
`kubectl port-forward` command, which forwards all traffic sent to a
local port (e.g., 8443) to the port of a remote service:
```sh
kubectl port-forward -n kubernetes-dashboard svc/kubernetes-dashboard 8443:443
```
Then, the Dashboard can be accessed through a browser at
<https://localhost:8443>.
