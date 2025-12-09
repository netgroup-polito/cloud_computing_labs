<table style="width:100%">
  <tr>
    <td align="left"><a href="../1.1/README.md">⬅️ Back to Sec 1.1</a></td>
    <td align="left"><a href="../README.md">⬅️ Back to Home</a></td>
  </tr>
</table>

# Appendix: Solutions

This appendix contains possible solutions to the different sections of
the laboratory. The provided solutions are not unique, and different
implementations are possible.
Feel free to adapt them to your needs, or implement your own version
from scratch.

> [!IMPORTANT]
> This appendix is provided only as a reference. It is strongly
> suggested to first try to implement the different sections on your own,
> before looking at the proposed solutions.
> Use this appendix only to check your implementation or get hints in
> case you are stuck.


## A.1 Packaging the website as a Docker image
A possible implementation of the `Dockerfile` packaging the website
is provided below.

```Dockerfile
# 1. Build stage - Generate the Hugo website
FROM alpine:3.18 AS builder
## 1.1 Install Hugo
RUN apk add --no-cache hugo
## 1.2 Copy the Hugo website source
WORKDIR /site
COPY . .
## 1.3 Build the website (output goes to /site/public)
RUN hugo -v

# 2. Runtime stage - Serve the website with nginx
FROM nginx:alpine
# 2.1 Copy the generated website from builder stage
COPY --from=builder /site/public /usr/share/nginx/html
# 2.2 Expose port 80 for HTTP traffic
EXPOSE 80
# 2.3 Start nginx server
CMD ["nginx", "-g", "daemon off;"]
```

The above Dockerfile leverages a **multi-stage build** approach.
In the *first stage*, based on an Alpine Linux image, Hugo is installed and
used to generate the static website files from the source content, just like
you would do on your local machine.

In the *second stage*, based on the official nginx image, the generated
website files are copied from the previous stage into the appropriate
directory served by nginx.
This results in a lightweight image containing only the necessary
components to serve the website (e.g., no Hugo or build tools are included).

Once the Dockerfile is ready, you can build the image as follows:
```sh
docker build -t <your-username>/website:v0.1 .
```
> [!WARNING]
> Remember to replace `<your-username>` with your actual Docker Hub username (or the appropriate registry URL if using Harbor).
> [Chapter 1](../1.1/README.md) contains additional information about tagging and pushing images using the correct format for the Crownlabs' Harbor image registry.

> [!NOTE]
> It is expected that the Dockerfile is placed in the root directory of the Hugo website (i.e., alongside the `config.toml` file) and that the build command is executed from the same location.

After the image is built, you can run it locally to verify that the
website is served correctly:
```sh
docker run -d -p 8080:80 <your-username>/website:v0.1
```

You can then access the website by navigating to `http://localhost:8080` in your web browser.
If everything works as expected, you can proceed to push the image to
your chosen public registry (Docker Hub or Harbor).


## A.2 Deploying the website on Kubernetes
A possible implementation of the Kubernetes *Deployment* manifest to
deploy the Docker image on the cluster is provided below using the *Harbor* registry with a student ID of `sXXXXXX`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: website
  labels:
    app: website
spec:
  replicas: 2 
  selector:
    matchLabels:
      app: website
  template:
    metadata:
      labels:
        app: website
    spec:
      containers:
      - name: website
        image: harbor.crownlabs.polito.it/cloud-sandbox/sXXXXXX/website:v0.1
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
```

> [!NOTE]
> This example uses a Deployment rather than a Pod directly, to ensure
> higher availability and scalability.

You also need a Service to expose the Deployment within the cluster:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: website
spec:
  selector:
    app: website
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
```

Finally, an Ingress resource is needed to expose the Service externally.
Crownlabs provides a pre-configured Ingress controller (`nginx-external`) and TLS certificates for sandbox namespaces (`crownlabs-sandbox-secret`).
Here is an example Ingress manifest:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: website
spec:
  ingressClassName: "nginx-external"
  tls:
  - hosts:
    - sXXXXXX.sandbox.crownlabs.polito.it
    secretName: crownlabs-sandbox-secret  # Pre-existing cert
  rules:
  - host: sXXXXXX.sandbox.crownlabs.polito.it
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: website
            port:
              number: 80
```

Since you will deploy the website on the CrownLabs Kubernetes cluster,
you need to obtain the kubeconfig file and configure your `kubectl` tool
to interact with your *sandbox namespace*. Refer to the [CrownLabs website](https://crownlabs.polito.it/resources/sandbox/) for detailed instructions. At the end, you will have a kubeconfig file ready to use. 

After creating the above resources using `kubectl apply -f <filename> --kubeconfig=<path-to-crownlabs-kubeconfig>`,
you should be able to access your deployed website at
`https://sXXXXXX.sandbox.crownlabs.polito.it`.

> [!NOTE]
> The Ingress will take some time to be fully operational.
> So, if you cannot access the website immediately, wait a few minutes and try again.

> [!WARNING]
> Given the self-signed nature of the provided TLS certificate, your browser might still show a warning.
> Just proceed to accept the "risk" and continue to the website.


