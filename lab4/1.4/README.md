<table style="width:100%">
  <tr>
    <td align="left"><a href="../1.4/README.md">⬅️ Previous</a></td>
    <td align="right"><a href="../README.md">End ✅</a></td>
  </tr>
</table>

# 4. Deploying the website on the Kubernetes cluster

With the Docker image pushed on a public repository, it is finally
possible to deploy the website on the student’s **sandbox namespace** on
the Kubernetes cluster.
The definition of the appropriate manifests is left to the reader, which should determine the types of resources (and their configuration) required to **deploy the website and make it accessible** to external clients.
To select the appropriate resource configuration, let consider the following requirements:

1.  the resulting webpage should be accessible at
    <https://sxxxxxx.sandbox.crownlabs.polito.it> (where `sxxxxxx`
    represents the student’s ID);

2.  the certificate of the website should be valid, to avoid warning
    messages from the browser; to simplify this task, each sandbox
    namespace already contains the `crownlabs-sandbox-secret` secret,
    including a trusted wildcard certificate;

3.  the Kubernetes nodes are characterized by private IP addresses, and
    LoadBalancer services cannot be used. However, CrownLabs provides a
    pre-configured Ingress controller (`nginx-external`) to expose services
    externally (through Ingress resources);

4.  (bonus) the application should survive the failure of a Kubernetes
    worker node, without notice from possible clients.