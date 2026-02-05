# Lab: Advanced Deployment Strategies (Helm & ArgoCD)

In previous labs, you managed Kubernetes resources by applying raw manifest files (YAML). As applications grow in complexity, managing hundreds of YAML files becomes inefficient. This lab introduces **Package Management** and **GitOps**.

**Objectives:**

1. Set up a local Kubernetes cluster using **Kind**.
2. Understand **Helm** to install and manage application packages.
3. Understand **ArgoCD** to implement GitOps automated delivery.

**Prerequisites:**

* **Docker** must be installed and running on your machine.
* `kubectl` installed.
* Internet access.

---

## Part 0: Environment Setup (k3s)

**k3s** is a tool for running a lightweight local Kubernetes clusters. It is lightweight and perfect for testing. In Crownlabs, start by creating a VM using the template "Cloud Computing: Kubernetes".

### 0.1 Install k3s and create the kubernetes cluster

Run the following commands to start your Kubernetes cluster.

1. Run the following command to install K3s:
    ```bash
    curl -sfL https://get.k3s.io | sh -
    ```

2. Export the kubeconfig path to work with \texttt{kubectl}
    ```bash
    export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
    ```
3. Change the permissions on the kubeconfig file
    ```bash
    sudo chmod 644 /etc/rancher/k3s/k3s.yaml
    ```
4. Verify the installation:
    ```bash
    kubectl get nodes
    ```
    
The server node should appear in the output.

---

## Part 1: Package Management with Helm

Helm is the **Package Manager for Kubernetes**.

Just as you use `apt` or `yum` to manage applications on Linux, or `brew` on macOS, you use Helm to manage applications on Kubernetes. It simplifies the process of defining, installing, and upgrading even the most complex Kubernetes applications.

### How it Works

Helm introduces three key concepts:

1. **Charts:** A Chart is a package. It is a directory containing a collection of files (YAML templates) that describe a related set of Kubernetes resources (Deployments, Services, Secrets, Ingress, etc.). Instead of hardcoding values, these files use variables.
2. **Values:** The `values.yaml` file allows you to define the default configuration for a Chart (e.g., which image to use, which port to open). You can override these values during installation to customize the app without changing the source code.
3. **Releases:** When you install a Chart into your cluster, a new instance is created called a **Release**. You can install the same Chart multiple times (e.g., one for `prod`, one for `dev`), and each will be a unique Release.

**The Workflow:**
When you run `helm install`, the Helm client takes the templates from the Chart, combines them with your configuration values, renders valid Kubernetes manifests, and sends them to the Kubernetes API to be created.

### 1.1 Install Helm

If you haven't installed Helm yet:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

```

Verify: `helm version`

### 1.2 Adding a Repository

Helm charts are stored in repositories. We will use the Bitnami repository to run a sinple web server.

1. Add the repo:
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami

```


2. Update your local cache:
```bash
helm repo update

```



### 1.3 Installing a Chart

We will install an NGINX web server. In Helm, a running instance of a chart is called a **Release**.

1. Create a namespace:
```bash
kubectl create namespace helm-lab

```


2. Install the chart:
```bash
helm install my-web-server bitnami/nginx \
  --namespace helm-lab \
  --set service.type=NodePort

```

**Verify the resources:**

```bash
kubectl get all -n helm-lab

```

You will see that Helm automatically created the Deployment, Service, and Pods.

If you now create in Crownlabs a VM from the template "Cloud Computing: Client VM" you can open the browser and connect to the newly created webserver connecting to *<Kubernetes_VM_IP>:<Nodeport_port>*

### 1.4 Upgrading a Release

Let's change the configuration (increase replicas) without editing raw YAML files.

1. Run the upgrade command:
```bash
helm upgrade my-web-server bitnami/nginx \
  --namespace helm-lab \
  --set replicaCount=3

```

2. Verify the pods (you should see 3 now):
```bash
kubectl get pods -n helm-lab

```

### 1.5 Rollback

If an upgrade breaks something, you can roll back instantly.

1. Roll back to Revision 1:
```bash
helm rollback my-web-server 1 -n helm-lab

```


2. Verify pods are back to 1:
```bash
kubectl get pods -n helm-lab

```

### 1.6 Clean Up (Helm)

```bash
helm uninstall my-web-server -n helm-lab

```

---

## Part 2: Continuous Delivery with ArgoCD

For the purpose of this lab, we will use as an example a simplified helm chart available [here](https://github.com/netgroup-polito/gitops_template_helm_chart). The repository hosts a Helm chart containing the definition of a deployment in Kubernetes.

ArgoCD ensures that the state of your cluster matches the state defined in a Git repository.

### 2.1 Install ArgoCD

1. Create the namespace:
```bash
kubectl create namespace argocd

```

2. Apply the official manifest:
```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

```

3. Wait for pods to be ready:
```bash
kubectl get pods -n argocd -w

```

### 2.2 Access the ArgoCD UI

Now that ArgoCD is running we can connect to the dashboard. First, we need to make the UI service of typo NodePort to make it accessible from aoutside the cluster by using the following:

1. **Open a new terminal** and run:
```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'

```

2. Open your browser in your client VM to `https://<Kubernetes_VM_IP>:<Nodeport_port>` (Accept the security warning).

### 2.3 Login

1. **User:** `admin`
2. **Password:** Get it from the secret:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

```

Now, there are two ways of configuring ArgoCD to sync with a remote repository:
1. The more straightforward option involves the interaction with the web-UI
2. The least straightforward is by means of configuration files.

In the following we will see the UI configuration.

### 2.4 Deploying application via UI

The steps to be performed are the following:

1. Start by creating what in the ArgoCD nomenclature is an *Application*

2. Specify configuration parameters as in the following picture. Provide a name for the application, a project name, and a sync policy. The sync policy can be either set to automating (i.e., meaning that ArgoCD will continously poll for updates in the GitHub repository) or to manual (i.e., you tell ArgoCD when to check for changes)
<p align="center">
  <img src="./Screenshot from 2026-02-05 16-28-59.png" alt="Physical setup: how to connect to your virtualized resources" width="80%">
</p>
<p align="center"><em></em></p>

3. Move on to the second part of the configuration, filling the textboxes with the following values. Specifically, what is needed is the url of the target repository, and a possible subpath for the chart.

<p align="center">
  <img src="./Screenshot from 2026-02-05 16-16-10.png" alt="Physical setup: how to connect to your virtualized resources" width="80%">
</p>
<p align="center"><em></em></p>

4. If configured correctly, ArgoCD will automatically detect the Helm chart, showing possible configuration options before doing the deployment. For the time being, leave them as they are, we'll modify them later.

5. Click on the create button on top to create the application. Now we only need to let ArgoCD synchronize with the remote repository and deploy everything in our cluster. You can check the pod running in the cluster by typing:
```bash
kubectl get pod -n demo
```

### 2.6 Testing Self-Healing

1. Check the ArgoCD UI; the app should be `Synced` and `Healthy`.
2. **Break the cluster:** Manually delete the deployment.
```bash
kubectl delete deployment test -n demo

```


3. **Watch the magic:** ArgoCD will detect the configuration drift (Cluster does not match Git) and immediately recreate the deployment.
4. Verify it's back:
```bash
kubectl get deployments

```
