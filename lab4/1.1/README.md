<table style="width:100%">
  <tr>
    <td align="left"><a href="../README.md">⬅️ Previous</a></td>
    <td align="right"><a href="../1.2/README.md">Next ➡️</a></td>
  </tr>
</table>

# 1. Physical setup
During this laboratory, we will leverage a different setup with respect to the previous exercises.
Specifically, we will use:

1.  A **management workstation**, to prepare the website, package it as
    a Docker image and interact with the target Kubernetes cluster
    (i.e., issue commands through `kubectl`). To this end, you can start
    the appropriate instance available in CrownLabs or, if you prefer,
    use your own laptop.

2.  A dedicated **slice of the CrownLabs Kubernetes cluster**, to host
    the deployed website and expose it to the external world. In
    particular, your CrownLabs account has been associated with what we
    call a *sandbox namespace*, that is a Kubernetes namespace you can
    freely access and use to deploy your own workloads. Additional
    information about how to configure your management workstation to
    interact with the sandbox namespace is available on the [CrownLabs
    website](https://crownlabs.polito.it/resources/sandbox/).
  > [!WARNING]
  > Given you are using a shared infrastructure, your account is associated with limited permissions.
  > In particular, you cannot access resources outside of your namespace, or cluster-wide resources (e.g., nodes).
  > Nonetheless, be respectful of others: do not attempt to perform operations which might damage other users or the CrownLabs infrastructure itself.

3.  A **public docker registry**, to publish the docker image containing
    the resulting website. You can leverage [Docker Hub](https://hub.docker.com/) (you need to
    either have or create a free account), or Harbor, a [registry made available by CrownLabs](https://harbor.crownlabs.polito.it/). 
    To access the Harbor dashboard, you need to use your CrownLabs credentials; then, click on your username (top right of the page), select the *User Profile* tab and copy the CLI
    secret: that is the password to use to login with the `docker login`
    command. To push images on Harbor, their tag shall be composed of
    the entire registry url, the `cloud-sandbox` repository and the
    student’s ID as image name prefix. For instance, a student with ID
    `s123098` could tag an image as
    `harbor.crownlabs.polito.it/cloud-sandbox/s123098/website:v0.1`
