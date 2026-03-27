> [!WARNING]
> This exercise requires the access to specific resources in Crownlabs. If you want to replicate the exact scenario please contact us at [stefano.galantino@polito.it](mailto:stefano.galantino@polito.it) and [jacopo.marino@polito.it](mailto:jacopo.marino@polito.it).

# Exercise 3 - DevOps in Kubernetes
The goal of this exercise is to set up an infrastructure with three environments (dev, test, and prod) and a management VM to control them.
The management VM will host Argo CD, which will deploy applications to the other VMs using GitOps principles.

By the end of the exercise, you will be able to observe automatic deployments to the correct environment clusters based on the actions you perform in the application repository, such as opening a pull request, merging changes, and creating a release.

Index:
- [Infrastructure](#infrastructure)
- [Demo Web Application](#demo-web-application)
- [CI/CD and GitOps](#cicd-and-gitops)
- [Test the Application Deployment Through the DevOps Pipeline](#test-the-application-deployment-through-the-devops-pipeline)

# Infrastructure
## Infrastructure Overview
### VM Provisioning
For this exercise, you need to provision **4 Virtual Machines (VMs)** using [**Crownlabs**](https://ng.crownlabs.polito.it), where the appropriate workspace is available from the main dashboard.

The required VMs are:
- **1 VM of type *Ubuntu Desktop 22.04 (Persistent)***, recommended label: `mgmt`
- **3 VMs of type *Ubuntu Server 22.04 (Persistent)***, recommended labels: `dev`, `test`, and `prod`

These VMs are created and managed entirely through the Crownlabs platform.

> [!WARNING]
> If you would like to connect to these VMs from your local machine, you need to upload your SSH public key to the CrownLabs dashboard **before creating the VMs**. Please follow the instructions provided [here](https://crownlabs.polito.it/resources/crownlabs_ssh/) to complete the setup.
> Please use the bastion `ssh.ng.crownlabs.polito.it` and not the default one `ssh.crownlabs.polito.it` in the ready SSH command, otherwise you will not be able to connect to the VMs.

To create a new VM, open the Crownlabs dashboard and navigate to the **workspace**. There, you will see the list of available VMs.
To create a new VM, click the **Create** button.

![crownlabs-create-vm.png](/exercise3/res/crownlabs-create-vm.png)

> [!NOTE]
> All VMs are created with the default username and password set to `crownlabs`.

> [!WARNING]
> **DO NOT TURN OFF VMs** to avoid IP changes, which means you need to setup the infrastructure again from scratch.

### Infrastructure layout
Once provisioned, the infrastructure is organized as follows:
- The **management VM (mgmt)**, equipped with a GUI, is used for central management and it hosts:
    - Kubernetes cluster
    - [**Argo CD](https://argo-cd.readthedocs.io/en/stable/)** (Continuous Deployment tool)
- The **dev**, **test**, and **prod** environment VMs, with just the terminal interface, are used for application deployment and they host:
    - Kubernetes clusters
    - Demo application at appropriate state (dev, test, or prod)

Each environment VM runs its own Kubernetes cluster to ensure proper isolation between development, testing, and production.

The Kubernetes distribution chosen for this Challenge is [**K3s**](https://k3s.io/), selected for its lightweight footprint, fast installation, and suitability for single-node clusters.

![logical-schema-infrastructure.drawio.png](/exercise3/res/logical-schema-infrastructure.drawio.png)

## Prerequisites for all VMs
Before proceeding with the setup, ensure that all VMs are up to date.
Run the following commands to update the package list and upgrade installed packages:
```bash
sudo apt update
sudo apt upgrade -y
```

Install the required dependencies needed for the next steps:
```bash
sudo apt install curl -y
```

These prerequisites must be completed on **all VMs** (mgmt, dev, test, and prod) before continuing with the installation steps.

### Change VM hostname
To simplify the management of virtual machines, it is useful to rename their hostnames using a convention that is easy for humans to read and understand.
To achieve this, we first define a consistent naming convention.
We use a structured hostname format that encodes the **location**, **environment**, and **instance number**:

```
<site><site-id>-<env><env-id>
```

Example:
```
trn01-dev01
```

To apply this naming convention, run the appropriate command on each VM to update its hostname accordingly:
```bash
sudo hostnamectl set-hostname trn01-dev01
sudo hostnamectl set-hostname trn01-test01
sudo hostnamectl set-hostname trn01-prod01
sudo hostnamectl set-hostname trn01-mgmt01
```

To make this change effective, you need to restart the VM:
```bash
sudo reboot
```

### Get VMs IP
To proceed with the setup, you must collect the **IP addresses assigned to the network interfaces of all VMs**.
Do not use the IPs shown in the Crownlabs dashboard, as those differ from the addresses assigned to VMs network interfaces.
On each VM, run the following command and copy the resulting IP address. Then, record the values in the table below.
```bash
ip -4 -o addr show enp1s0 | awk '{print $4}' | cut -d/ -f1
```

The IP addresses shown in the table are **examples only** and must be replaced with the actual values from your environment.
| **VM** |    **IP**     |
| ------ | ------------- |
| `mgmt` | 172.23.22.112 |
| `dev`  | 172.23.22.212 |
| `test` | 172.23.22.48  |
| `prod` | 172.23.22.98  |

## Management VM Setup
This section describes how to configure the **management (mgmt) VM**, which acts as the control plane for the entire infrastructure.
On this VM, you will install a Kubernetes cluster and deploy the Continuous Deployment tool **Argo CD**, which will be used to manage and synchronize applications across the different environments.
The following subsections guide you step by step through the installation and configuration process required to prepare the management VM.

### Re-install Firefox
If the default browser fails to open correctly, re-install **Firefox** using the following commands:
```bash
sudo snap remove firefox

sudo snap remove snapd
sudo apt purge snapd

sudo add-apt-repository ppa:mozillateam/ppa -y
sudo apt update
sudo apt install firefox-esr
```

### Install K3s
To install **K3s**, run the following command:
```bash
curl -sfL https://get.k3s.io | K3S_KUBECONFIG_MODE="644" sh -
```

This command downloads and installs K3s, and configures the `kubeconfig` file to be readable by non-root users.
Wait a few minutes for the installation to complete.
Once the installation finishes, verify that the cluster is running by listing all pods:
```bash
kubectl get pods -A
```

You should see output similar to the following, showing system pods in the `kube-system` namespace running successfully.
```
NAMESPACE     NAME                                      READY   STATUS      RESTARTS   AGE
kube-system   coredns-7f496c8d7d-65sv6                  1/1     Running     0          2m57s
kube-system   helm-install-traefik-crd-2mjzt            0/1     Completed   0          2m58s
kube-system   helm-install-traefik-hw89q                0/1     Completed   1          2m58s
kube-system   local-path-provisioner-578895bd58-s8v8z   1/1     Running     0          2m57s
kube-system   metrics-server-7b9c9c4b9c-fv7pp           1/1     Running     0          2m57s
kube-system   svclb-traefik-247c8c92-cw9tr              2/2     Running     0          2m34s
kube-system   traefik-6f5f87584-wq2tq                   1/1     Running     0          2m34s
```

If the output is similar, then the K3s cluster has been successfully installed.

### Install Argo CD

> [!NOTE]
> This section is based on the official [Argo CD documentation](https://argo-cd.readthedocs.io/en/stable/getting_started/).

To install Argo CD, run the following commands.
The first command creates a dedicated namespace, and the second applies the Argo CD installation manifest to the cluster:
```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

#### Verify the installation
Check that Argo CD has been installed correctly by listing the pods in the `argocd` namespace:
```bash
kubectl get pod -n argocd
```

You should see output similar to the following:
```
NAME                                                READY   STATUS    RESTARTS      AGE
argocd-application-controller-0                     1/1     Running   0             71s
argocd-applicationset-controller-5796dcfc94-nx79c   1/1     Running   0             71s
argocd-dex-server-6d57f6f6b6-ghxjj                  1/1     Running   2 (44s ago)   71s
argocd-notifications-controller-bb4f97f47-vgzcn     1/1     Running   0             71s
argocd-redis-6dfddccb76-7svj7                       1/1     Running   0             71s
argocd-repo-server-77d887cfb9-jgvdt                 1/1     Running   0             71s
argocd-server-7fb8c5f74-b69zk                       1/1     Running   0             71s
```

Wait until **all pods are in Running state**. You may need to run the command multiple times while the cluster initializes.

#### Expose the Argo CD UI via NodePort
By default, the Argo CD server is exposed as a `ClusterIP` service.
To access the UI from your VM, we will patch it to a `NodePort`.
First, list all services in the `argocd` namespace:
```bash
kubectl get svc -n argocd
```

The service we need to modify is `argocd-server`.
Patch the service to change its type to `NodePort` and expose HTTPS on port `30443`:
```bash
kubectl patch svc argocd-server -n argocd --type='merge' -p '{
  "spec": {
    "type": "NodePort",
    "ports": [
      {
        "name": "http",
        "port": 80,
        "targetPort": 8080
      },
      {
        "name": "https",
        "port": 443,
        "targetPort": 8080,
        "nodePort": 30443
      }
    ]
  }
}'
```

If the patch is applied successfully, you should see:
```bash
service/argocd-server patched
```

Verify that the service is now exposed as a `NodePort`:
```bash
kubectl get svc argocd-server -n argocd
```

Expected output:
```
NAME            TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
argocd-server   NodePort   10.43.140.82   <none>        80:3XXXX/TCP,443:30443/TCP   5m4s
```

#### Access the Argo CD UI
Open a browser **on the VM** and navigate to:
```bash
https://localhost:30443
```

Ignore the browser security warning and continue.
You should now see the Argo CD login page.

#### Retrieve the admin password

Argo CD generates an initial admin password automatically.
Retrieve it using the following command:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

Login with:
- **Username:** `admin`
- **Password:** `the value retrieved above`

Once logged in, you should see the Argo CD dashboard similar to the following:

![argocd-dashboard.png](/exercise3/res/argocd-dashboard.png)

> [!NOTE]
> For ease of use, we recommend storing the password in the Firefox password manager.

#### Argo CD CLI
For later use, install the Argo CD CLI now:
```bash
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

Now, log in to Argo CD using the CLI:
```bash
argocd login localhost:30443 --insecure --username admin --password <the_value_retrieved_above>
```

Expected output:
```
'admin:login' logged in successfully
Context 'localhost:30443' updated
```

### [OPTIONAL] Create a `kubectl` alias
To avoid typing `kubectl` every time, you can create a convenient alias.
First, make sure you are in your home directory (if you're unsure, run the first command):
```bash
cd ~
echo "alias k='kubectl'" >> ~/.bashrc
source .bashrc
```

After this, you can use `k` instead of `kubectl`:
```bash
# List all pods in all namespaces
k get pods -A

# List services in the current namespace
k get svc

# Describe a specific pod
k describe pod my-pod-name

# Check cluster nodes
k get nodes
```

You can use the `k` alias anywhere you would normally use `kubectl`.

### [OPTIONAL] Enable `kubectl` autocompletion
You can also enable **command autocompletion** for `kubectl`, including the `k` alias.
Again, ensure you are in your home directory:
```bash
cd ~
kubectl completion bash >> ~/.bashrc
echo "complete -F __start_kubectl k" >> ~/.bashrc
source .bashrc
```

Once enabled, you can press **Tab** to auto-complete commands, resource names, and flags, for example:
```bash
k get po<Tab>
```

### [OPTIONAL] Install K9s
To simplify later operations and make it easier to inspect deployments, you can install a CLI tool called [K9s](https://k9scli.io). K9s provides an interactive, terminal-based interface for managing Kubernetes clusters, offering a more user-friendly experience compared to using `kubectl` alone. 

You only need to install K9s on the **management VM**, since you can switch between Kubernetes contexts to manage all clusters (`mgmt`, `dev`, `test`, and `prod`). Refer to Section [*Dev, Test, and Prod VMs Setup*](#dev-test-and-prod-vms-setup) for instructions on how to add and configure multiple contexts on the management VM.

```bash
curl -Lo k9s.tar.gz https://github.com/derailed/k9s/releases/latest/download/k9s_Linux_amd64.tar.gz
tar -xzf k9s.tar.gz
sudo mv k9s /usr/local/bin/
rm k9s.tar.gz LICENSE README.md
```

Verify that K9s has been installed correctly by launching it:
```bash
k9s
```

If the interface starts successfully, the installation is complete.
To exit K9s, press `Ctrl + C`

### [OPTIONAL] Port forward Argo CD interface to your local machine
#### Get Management VM IP via Crownlabs Dashboard
To proceed with this, you need to collect the **IP addresses of management VM**.
1. Go to the [**Crownlabs**](https://ng.crownlabs.polito.it) dashboard.
2. Open the **Active** tab.
3. You should see a dashboard similar to the one shown below.
4. On management VM, click the **ⓘ (information) icon** (highlighted with a red circle in the figure).
5. Copy the **IP address** shown for the VM and store it for later use.

![crownlabs-get-vm-ip.png](/exercise3/res/crownlabs-get-vm-ip.png)

#### Port forwarding from the management VM to your local machine
If you want to open the Argo CD interface on your local machine, you can forward the port over SSH.
To do this, run the following command in a terminal on your local machine (insert VM password `crownlabs`):
```bash
ssh -L 30443:localhost:30443 \
    -J bastion@ssh.ng.crownlabs.polito.it \
    crownlabs@<mgmt_vm_ip>
```

- `-L 30443:localhost:30443`
    - **local port**: 30443
    - **remote host** (from the remote machine's POV): localhost
    - **remote port**: 30443
- `-J bastion@ssh.ng.crownlabs.polito.it`
    - Use the bastion as a jump host
- `crownlabs@<mgmt_vm_ip>`
    - Final destination machine

## Dev, Test, and Prod VMs Setup
This section describes how to configure the **environment-specific VMs**: **dev**, **test**, and **prod**.
Each of these VMs hosts its own dedicated Kubernetes cluster, fully isolated from the others, and is intended to represent a distinct stage of the application lifecycle.
On each VM, you will install a Kubernetes cluster using **K3s**.
These clusters will be managed remotely from the management VM through **Argo CD**, enabling a GitOps-based workflow for application deployment and synchronization.
The setup steps in this section must be performed **independently on each VM** (dev, test, and prod) unless explicitly stated otherwise.

To install K3s on the dev, test, and prod VMs, run the same installation command used for the management VM, as described in the [**Install** **K3s**](#install-k3s) section.

### Configure SSH access from the Management VM
Before proceeding with the next steps, you need to configure **passwordless SSH access** from the **management (mgmt) VM** to the **environment VMs** (dev, test, and prod).
This setup allows you to:
- Log in via SSH using **key-based authentication**
- Avoid repeatedly entering usernames and passwords
- Simplify later automation and configuration steps

#### Step 1: Generate an SSH key pair on the management VM
On the **management VM**, generate a new SSH key pair:
```bash
ssh-keygen -t ed25519
```

When prompted, you can press **Enter** to accept the default file location and leave the passphrase empty.
Verify that the keys have been created:
```bash
ls ~/.ssh
```

You should see output similar to:
```bash
authorized_keys  id_ed25519  id_ed25519.pub
```

#### Step 2: Copy the public key to the environment VMs
Next, you need to copy the **public key** from the management VM to each environment VM. This authorizes the management VM to connect via SSH without requiring a password.
First, define the SSH user and the IP addresses of the environment VMs (use the IPs collected from the **Crownlabs** dashboard):
```bash
USER=crownlabs
DEV_VM=<ip_dev_vm>
TEST_VM=<ip_test_vm>
PROD_VM=<ip_prod_vm>
```

Now, copy the SSH key to each VM:
```bash
ssh-copy-id $USER@$DEV_VM
ssh-copy-id $USER@$TEST_VM
ssh-copy-id $USER@$PROD_VM
```

When prompted, enter the password for the environment VMs: `crownlabs`.
The **first time** you connect to an environment VM from the management VM, you will see a prompt similar to the following:
```
The authenticity of host 'X.X.X.X (X.X.X.X)' can't be established.
ED25519 key fingerprint is SHA256:...
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

Type `yes` and press **Enter**. This prompt will only appear once per VM.

#### Step 3: Verify passwordless SSH access
After the key has been copied, you should be able to connect to the environment VMs **without being asked for a password**.
Test the connection (one command at a time, then `exit` after each command):
```bash
ssh $USER@$DEV_VM
ssh $USER@$TEST_VM
ssh $USER@$PROD_VM
```

If the connection is successful and you are logged into the environment VM, the SSH setup is working correctly. You can now exit the SSH session by running:
```bash
exit
```

This will return you to the management VM shell.

### Retrieve `kubeconfig` from environment VMs to management VM
On the **management VM**, run the following commands to retrieve the `kubeconfig` files from dev, test, and prod, and merge them (together with the mgmt `kubeconfig`) into a **single `kubeconfig` file**.
Create a folder to store `kubeconfig` files:
```bash
mkdir -p ~/kubeconfigs
```

Define SSH user and VM IPs, using the IP addresses collected from the **Crownlabs** dashboard:
```bash
USER=crownlabs
DEV_VM=<ip_dev_vm>
TEST_VM=<ip_test_vm>
PROD_VM=<ip_prod_vm>
```

Copy `kubeconfig` files from dev/test/prod to mgmt:
```bash
scp $USER@$DEV_VM:/etc/rancher/k3s/k3s.yaml ~/kubeconfigs/dev.yaml
scp $USER@$TEST_VM:/etc/rancher/k3s/k3s.yaml ~/kubeconfigs/test.yaml
scp $USER@$PROD_VM:/etc/rancher/k3s/k3s.yaml ~/kubeconfigs/prod.yaml
```

Copy the mgmt `kubeconfig` into the same folder :
```bash
cp /etc/rancher/k3s/k3s.yaml ~/kubeconfigs/mgmt.yaml
```

Verify that all `kubeconfig` files are present:
```bash
ls ~/kubeconfigs
```

Expected output:
```bash
dev.yaml  mgmt.yaml  test.yaml  prod.yaml
```

By default, K3s assigns the same names to the **cluster**, **user**, and **context** (commonly `default`) on every cluster, which can lead to conflicts when multiple `kubeconfig` files are combined.
Renaming these entries ensures each cluster is clearly identifiable.
In addition, K3s configures the Kubernetes API server address as `https://127.0.0.1:6443`, which is only reachable from within the VM where the cluster is running.
Since these `kubeconfig` files will be used from the management VM, the API server address must be updated to use the corresponding VM IP instead.

Create a Python script to manage this with the following commands:
```bash
cd ~
nano fix_kubeconfigs.py
```

Now, copy the following code inside the opened file, modifying the required fields (`DEV_VM`, `TEST_VM`, `PROD_VM`):
```python
#!/usr/bin/env python3
import os
import yaml

# ================== VARIABLES (EDIT THESE) ==================
KUBECONFIG_DIR = os.path.expanduser("~/kubeconfigs")

DEV_VM  = "<ip_dev_vm>"
TEST_VM = "<ip_test_vm>"
PROD_VM = "<ip_prod_vm>"
# ============================================================

CONFIGS = [
    # filename, cluster_name, user_name, context_name, server_ip
    ("dev.yaml",  "dev",  "dev-user",  "dev",  DEV_VM),
    ("test.yaml", "test", "test-user", "test", TEST_VM),
    ("prod.yaml", "prod", "prod-user", "prod", PROD_VM),
    ("mgmt.yaml", "mgmt", "mgmt-user", "mgmt", "127.0.0.1"),  # keep mgmt local
]

def update_kubeconfig(path: str, cluster_new: str, user_new: str, context_new: str, server_ip: str) -> None:
    with open(path, "r") as f:
        cfg = yaml.safe_load(f)

    # Assumes a typical K3s kubeconfig: 1 cluster, 1 user, 1 context.
    old_context = cfg["contexts"][0]["name"]

    # Update server address (needed for env VMs)
    if server_ip != "127.0.0.1":
        cfg["clusters"][0]["cluster"]["server"] = f"https://{server_ip}:6443"

    # Rename cluster/user/context
    cfg["clusters"][0]["name"] = cluster_new
    cfg["users"][0]["name"] = user_new
    cfg["contexts"][0]["name"] = context_new

    # Rewire context references
    cfg["contexts"][0]["context"]["cluster"] = cluster_new
    cfg["contexts"][0]["context"]["user"] = user_new

    # Update current-context if needed
    if cfg.get("current-context") == old_context:
        cfg["current-context"] = context_new

    with open(path, "w") as f:
        yaml.safe_dump(cfg, f, default_flow_style=False, sort_keys=False)

def main():
    for fname, cluster_new, user_new, context_new, server_ip in CONFIGS:
        path = os.path.join(KUBECONFIG_DIR, fname)
        if not os.path.exists(path):
            raise FileNotFoundError(f"Missing file: {path}")

        print(f"Processing {path} ...")
        update_kubeconfig(path, cluster_new, user_new, context_new, server_ip)
        print(f"✔ Updated {fname}")

    print("\nAll kubeconfigs updated successfully.")

if __name__ == "__main__":
    main()
```

Save and exit, in nano is `Ctrl + O` → `Enter` → `Ctrl + X`
Now, run the script:
```bash
python3 fix_kubeconfigs.py
```

Create the `.kube` directory (if it does not exist):
```bash
mkdir -p ~/.kube
```

Merge all `kubeconfig` files into `~/.kube/config`:
```bash
KUBECONFIG=~/kubeconfigs/mgmt.yaml:~/kubeconfigs/dev.yaml:~/kubeconfigs/test.yaml:~/kubeconfigs/prod.yaml kubectl config view --merge --flatten > ~/.kube/config
```

Configure `kubectl` to always use the merged `kubeconfig`:
```bash
echo 'export KUBECONFIG=$HOME/.kube/config' >> ~/.bashrc
source ~/.bashrc
```

Verify the environment variable:
```bash
echo $KUBECONFIG
```

Expected output:
```
/home/crownlabs/.kube/config
```

Verify contexts are available:
```bash
kubectl config get-contexts
```

You should see at least these contexts:
```
CURRENT   NAME   CLUSTER   AUTHINFO    NAMESPACE
          dev    dev       dev-user
*         mgmt   mgmt      mgmt-user
          prod   prod      prod-user
          test   test      test-user
```

Test all contexts:
```bash
kubectl config use-context mgmt
kubectl get nodes

kubectl config use-context dev
kubectl get nodes

kubectl config use-context test
kubectl get nodes

kubectl config use-context prod
kubectl get nodes
```

If all commands return the nodes for the selected cluster, the `kubeconfig` merge is complete and working correctly.
Now, go back to the `mgmt` context:
```bash
kubectl config use-context mgmt
```

## Configure Argo CD
Now that all virtual machines are set up, the next step is to allow the **management VM** to control the other VMs using Argo CD.
This is done by leveraging the Kubernetes configuration prepared in the previous steps.
Open a terminal on the management VM and run the following commands.
These commands will register the **three environment clusters** (`dev`, `test`, and `prod`) with Argo CD.
```bash
argocd cluster add dev -y
argocd cluster add test -y
argocd cluster add prod -y
```

Now, verify that all clusters have been successfully added to Argo CD by running:
```bash
argocd cluster list
```

Expected output:
```
SERVER                          NAME        VERSION  STATUS      MESSAGE  PROJECT
https://XX.XX.XX.XX:6443        dev         1.34     Successful           
https://XX.XX.XX.XX:6443        test        1.34     Successful           
https://XX.XX.XX.XX:6443        prod        1.34     Successful           
https://kubernetes.default.svc  in-cluster            
```

> [!NOTE]
> Don't worry if the status is `Unknown`. At this stage, no application has been deployed yet, as indicated by the message: *"Cluster has no applications and is not being monitored."*

You can also verify this directly from the **Argo CD UI**
1. Open the Argo CD UI at `https://localhost:30443`
2. Log in to the dashboard
3. In the left sidebar, click **Settings**
4. Select **Clusters**
5. You should now see the **management cluster (mgmt)** along with the three environment clusters (**dev**, **test**, and **prod**).

This confirms that all clusters have been successfully registered with Argo CD.

# Demo Web Application
The demo application is a **simple single-page web application** designed to showcase the CI/CD pipeline.

It contains a **single button**, whose color will be changed to demonstrate how updates flow through the pipeline from commit to deployment.

![demo-app.png](/exercise3/res/demo-app.png)

To modify the button color, edit the `className` property of the Button component in `App.jsx`.
The color is defined using Tailwind CSS utility classes, specifically `bg-<color>-600` for the default state and `hover:bg-<color>-700` for the hover state. You can refer to the [Tailwind color documentation](https://tailwindcss.com/docs/colors).
Changing these values will result in a visible update once the pipeline runs and the application is redeployed.

```bash
<Button className="bg-blue-600 text-white transition active:scale-95 active:shadow-inner hover:bg-blue-700">
  Demo Button
</Button>
```

The code is available here: https://github.com/bl4ckj4c/challenge-app

# CI/CD and GitOps

> [!NOTE]
> As students, you are eligible to request the **GitHub Pro** plan for free. We strongly recommend applying for it, as it provides more GitHub Actions minutes compared to the standard free plan.

## Overview
### Goal
We will build a simple web application and demonstrate a realistic company-style delivery pipeline using:
- **2 Git repositories** (application code + GitOps deployments)
- **GitHub Actions** for CI and promotion logic
- **Argo CD** for GitOps deployments (pull-based)
- **Three workload clusters**: dev, test, prod (plus one management cluster hosting Argo CD)

This demo shows how code moves from feature branches to test and then production with strong controls.

### Repositories
#### Application Repository (App Repo)
Contains:
- Application source code (simple web UI)
- Dockerfile
- Helm Chart
- Unit tests/lint checks
- GitHub Actions workflows to build/test/publish images
- Logic to update the GitOps repo (via PRs)

This repo is responsible for producing deployable artifacts (container images and Helm charts).

#### GitOps Repository (Deployment Repo)
Contains:
- Helm values
- Argo CD resources (ApplicationSet + Applications)
- Environment configuration:
    - dev: ephemeral namespaces per branch
    - test: fixed namespace `challenge-test`
    - prod: fixed namespace `challenge-prod`

This repo is the **single source of truth** for what is deployed in each environment.

**Argo CD watches this repo and reconciles cluster state.**

### Clusters and Responsibilities

| **Cluster** | **Responsibilities** |
| --- | --- |
| Mgmt Cluster | Runs **Argo CD**; Argo CD has access to dev/test/prod clusters |
| Dev Cluster | Hosts ephemeral environments: one namespace per Pull Request (e.g., `challenge-pr-1`); updated on every Pull Request to main |
| Test Cluster | Hosts the testing environment, namespace: `challenge-test`; updated on every merge into main |
| Prod Cluster | Hosts the production environment, namespace: `challenge-prod`; updated only by a Release process |

### Deployment Model: Pull-based GitOps
**Important rule:** GitHub Actions does not deploy directly with `kubectl apply`.
Instead:
1. GitHub Actions runs tests/builds images
2. GitHub Actions updates the GitOps repo (creates a PR or commits changes)
3. Argo CD detects changes in GitOps repo and deploys automatically

This matches the “GitOps” model used in many organizations.

### Dev Workflow (Feature Branches → Dev Cluster)
#### Trigger
Every Pull Request (PR) to `main` branch

#### Actions
1. Run CI checks:
    - Lint
    - Unit tests
2. Build and push container image to registry
3. Push Helm artifacts to registry
4. Update the GitOps repo

**Namespace convention:** `challenge-pr-<pr_number>`

#### Argo CD mechanism
An **Argo CD ApplicationSet** generates an Argo CD Application per branch environment.
Result: each branch gets its own isolated environment automatically.

### Test Workflow (Merge to main → Test Cluster)
#### Trigger
Pull Request merged into `main`

#### Actions
1. Run CI checks again on main (same or stronger checks)
2. Build and push container image
3. Update the GitOps repo test configuration to deploy to:
    - Cluster: `test`
    - Namespace: `challenge-test`
4. Argo CD deploys automatically

Result: main always represents the current “integration-tested” version in test.

### Prod Workflow (Release → Prod Cluster)
#### Trigger
A new GitHub Release (or tag) is created

#### Hard requirement: Release must come from main
Before deploying:
- The pipeline verifies that the **tagged commit is reachable from main**
- If not, the pipeline fails and production deployment is blocked

This prevents deploying code that was never merged into main.

### Immutable Artifact Promotion (Digest-based deployment)

For production we promote **immutable images**:
- We deploy by **digest**, e.g. myapp@sha256:...
- We do not deploy latest or other mutable tags

Why:
- Ensures test and prod run the **exact same artifact**
- Prevents “tag drift” (a tag pointing to different images over time)
- Improves traceability and auditability

### Summary of Environment Rules

- **Dev**: many namespaces, one per PR (ephemeral)
- **Test**: single namespace `challenge-test`, updated on merge to `main`
- **Prod**: single namespace `challenge-prod`, updated only on Release
- **Argo CD** is always the deployer; GitHub Actions only changes Git state in GitOps repo
- Production deploys only if:
    1. Release commit is on `main`
    2. Deployment uses an immutable image digest

## Repositories setup
To continue, create **two GitHub repositories**:
- one for the **application code**
- one for the **GitOps configuration**

Make sure **at least the application repository is public** to avoid issues when pulling container images.

> [!WARNING]
> Please do not fork the repositories used for the demo. Instead, clone them and push the code to a new repository under your own GitHub account, as forking may cause issues with GitHub Actions permissions and workflow execution.

#### Enable GitHub Actions package permissions
To allow GitHub Actions to build and push container images, you must enable write permissions for `GITHUB_TOKEN`.
In the **application repository**:
1. Go to **Settings → Actions → General**
2. Under **Workflow permissions**, select **Read and write permissions**
3. Click **Save**

This allows `GITHUB_TOKEN` to publish packages.

#### Allow the application repo to push to the GitOps repo
To enable the application repository to update the GitOps repository, create a **fine-grained Personal Access Token (PAT)** with:
- **Contents: Read & Write** access
- Scope limited to the **GitOps repository**

Store this token as a secret in the **application repository** with the name `GITOPS_TOKEN`.
1. In GitHub, click your profile photo → **Settings**
2. Left sidebar → **Developer settings**
3. **Personal access tokens** → **Fine-grained tokens** → **Generate new token**
4. Set:
    - **Resource owner**: your user or org that owns the **GitOps repo**
    - **Expiration**: `31/07/2026`
    - **Repository access**
        - Select **Only select repositories**
        - Choose *only* the GitOps repo
    - **Repository permissions**
        - **Contents** → **Read and write** (required to push commits)
5. Click **Generate token** and **copy it immediately** (this is the only time it will be shown)
6. In the **application repository**, go to **Settings** → **Secrets and variables** → **Actions** → **New repository secret**
7. Set
    - **Name**: `GITOPS_TOKEN`
    - **Value**: paste the generated PAT
8. Click **Save**

> [!NOTE]
> You generally cannot push to a different repo using only GITHUB_TOKEN, so a PAT (or GitHub App) is the normal approach for cross-repo GitOps updates.

Also define these as variables to application repository:
- In the **application repository**, go to **Settings** → **Secrets and variables** → **Actions** → Variables → **New repository variable**
    - `GITOPS_REPO` like `my-org/my-gitops-repo`
    - `APP_NAME` like `challenge-app` 

## Argo CD CR
To allow Argo CD to understand what it needs to deploy, we must create and apply a Custom Resource (CR) that instructs Argo CD to watch a specific Helm chart and its corresponding configuration.
In this setup, Argo CD must be configured to use two sources:
- one source for the Helm chart, and
- one source for the values.yaml file, which defines the image tag to be used in the deployment.

Below are the three YAML files required to configure Argo CD to deploy the application to the corresponding cluster. Make sure to replace each `repoURL` with your own repository URL and adjust the manifests to match your final repository structure.

> [!NOTE]
> The Argo CD manifests must be applied to the cluster running in the management VM.

### Dev cluster
To deploy applications to the dev cluster, create and open the `manifest-dev.yaml` file using the following command:
```bash
nano manifest-dev.yaml
```

Paste the following content inside:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: challenge-dev
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/bl4ckj4c/challenge-gitops
        revision: main
        directories:
          - path: environments/dev/pr-*
  template:
    metadata:
      name: challenge-dev-{{path.basename}}
      finalizers:
        - resources-finalizer.argocd.argoproj.io
    spec:
      project: default
      destination:
        name: dev
        namespace: "{{path.basename}}"
      sources:
        - repoURL: ghcr.io/bl4ckj4c
          targetRevision: 0.1.1
          chart: challenge-app/challenge-app-chart
          helm:
            valueFiles:
              - $values/{{path}}/values.yaml
        - repoURL: https://github.com/bl4ckj4c/challenge-gitops
          targetRevision: main
          ref: values
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

Apply the manifest:
```bash
kubectl apply -f manifest-dev.yaml
```

### Test cluster
To deploy applications to the test cluster, create and open the `manifest-test.yaml` file using the following command:
```bash
nano manifest-test.yaml
```

Paste the following content inside:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: challenge-test
  namespace: argocd
spec:
  destination:
    name: test
    namespace: challenge-app
  project: default
  sources:
  - repoURL: ghcr.io/bl4ckj4c
    targetRevision: 0.1.1
    chart: challenge-app/challenge-app-chart
    helm:
      valueFiles:
        - $values/environments/test/values.yaml
  - repoURL: https://github.com/bl4ckj4c/challenge-gitops
    targetRevision: main
    ref: values
  syncPolicy:
    automated:
      enabled: true
      prune: true
      selfHeal: true
```

Apply the manifest:
```bash
kubectl apply -f manifest-test.yaml
```

### Prod cluster
To deploy applications to the prod cluster, create and open the `manifest-prod.yaml` file using the following command:
```bash
nano manifest-prod.yaml
```

Paste the following content inside:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: challenge-prod
  namespace: argocd
spec:
  destination:
    name: prod
    namespace: challenge-app
  project: default
  sources:
  - repoURL: ghcr.io/bl4ckj4c
    targetRevision: 0.1.1
    chart: challenge-app/challenge-app-chart
    helm:
      valueFiles:
        - $values/environments/prod/values.yaml
  - repoURL: https://github.com/bl4ckj4c/challenge-gitops
    targetRevision: main
    ref: values
  syncPolicy:
    automated:
      enabled: true
      prune: true
      selfHeal: true
```

Apply the manifest:
```bash
kubectl apply -f manifest-prod.yaml
```

## K8s ImagePullBackOff error
If you encounter an **ImagePullBackOff** error while deploying your application, make sure that your container image on **GHCR** is set to **public visibility**.
After pushing your first image, follow these steps:
1. Go to your repository on GitHub.
2. Click on **Packages** (right sidebar).
3. Select the image of your application.
4. Click on **Package settings** (right sidebar).
5. Scroll down to the **Danger Zone** section.
6. Click **Change visibility**.
7. Set the package to **Public** and confirm.

Once the image is public, Kubernetes should be able to pull it without authentication errors.

# Test the Application Deployment Through the DevOps Pipeline
To verify that everything works correctly, we will test each stage of the deployment process by:
	1.	Deploying the application to the dev cluster (via Pull Request)
	2.	Promoting it to test (after merge)
	3.	Promoting it to prod (after release)

To simulate a change, we will modify the button color as explained in the [Demo Web Application](#demo-web-application) section.

## Access the Management VM
First, open Crownlabs and access the GUI of the management VM.

Depending on previous attempts, the clusters may already have some applications deployed (for example, if you previously created a PR, merged it, or created a release).

Now we will trigger all the stages of the DevOps pipeline step by step.

## Clone the Application Repository
If you have not already done so, clone the application repository using the CLI (commands below) or using an IDE such as VSCode (recommended).
```bash
git clone https://github.com/bl4ckj4c/challenge-app.git
cd challenge-app
```

## Create a Feature Branch
Create a new branch, for example feature/button-color, and switch to it:
```bash
# Create the branch and switch to it
git checkout -b feature/button-color

# Push the branch to the remote repository
git push -u origin feature/button-color

# Verify that you are on the new branch
git branch
```

## Modify the Button Color
Change the button color in the application code.
You can refer to the [Tailwind color documentation](https://tailwindcss.com/docs/colors).
Modify only the color name in the relevant class.

## Create a Pull Request (Deploy to Dev)
Go to GitHub and create a Pull Request (PR) from: `feature/button-color → main`.
Once the CI/CD pipeline finishes:
- Open Argo CD
- You should see a new application named `challenge-dev-pr-<pr_number>` (where <pr_number> is the PR number)

On the application card you should see:
- Destination cluster: `dev`
- Namespace: `pr-<pr_number>`
- Status: `Healthy`
- Sync status: `Synced`

## Access the Dev Deployment
Switch to the dev Kubernetes context:
```bash
kubectl config use-context dev
```

Get the NodePort:
```bash
kubectl get svc -n pr-<pr_number>
```

Example output:
```
NAME                 TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
challenge-app-pr-6   NodePort   10.43.43.96     <none>        80:3XXXX/TCP   6d19h
```

Now open Firefox and navigate to `https://<dev_vm_ip>:3XXXX`. You should see the application running in the dev cluster with the new button color.

## Merge the PR (Deploy to Test)
If everything looks correct, merge the PR into main.

In GitHub:
1. Go to Pull Requests
2. Open your PR
3. Click Merge

After the merge completes, Argo CD will automatically deploy the new version to the test cluster.
You should see an application named `challenge-test` with status `Healthy` and `Synced`.

## Access the Test Deployment
Switch to the test context and get the NodePort:
```bash
kubectl config use-context test
kubectl get svc -n challenge-app
```

Example output:
```
NAME            TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
challenge-app   NodePort   10.43.174.153    <none>        80:3XXXX/TCP   6d20h
```

Now naviagte to `https://<test_vm_ip>:3XXXX`. You should see the updated button color deployed in the test cluster.

## Create a Release (Deploy to Production)
Now we promote the application to production.

On the GitHub application repository:
1. Click Releases (right sidebar)
2. Click Draft a new release
3. Create or select a tag
4. Add a title (e.g., Button <color>)
5. Click Publish release

After publishing the release, Argo CD will automatically deploy the new version to the prod cluster.
You should see `challenge-prod` with status `Healthy` and `Synced`.

## Access the Production Deployment
Switch to the prod context and get the NodePort:
```bash
kubectl config use-context prod
kubectl get svc -n challenge-app
```

Example output:
```
NAME            TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
challenge-app   NodePort   10.43.221.132    <none>        80:3XXXX/TCP   6d19h
```

Now navigate to `https://<prod_vm_ip>:3XXXX`. If you see the application with the updated button color running in production, congratulations!

You have successfully tested the complete DevOps pipeline:
- Code change
- Pull Request → Dev deployment
- Merge → Test deployment
- Release → Production deployment
