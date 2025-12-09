<table style="width:100%">
  <tr>
    <td align="left"><a href="../1.2/README.md">⬅️ Previous</a></td>
    <td align="right"><a href="../1.4/README.md">Next ➡️</a></td>
  </tr>
</table>

# 3. Packaging the website as a Docker image

It is now time to prepare the **Docker image** packaging the website
created in the previous section, so that it can be later deployed on a
Kubernetes cluster. To this end, it is necessary to create the
appropriate `Dockerfile` (within the root directory of the website).
Overall, the instructions in the Dockerfile shall perform two main
actions:

1.  **generate the website files** to be published (by means of the `hugo -v` command);

2.  **serve the resulting content** (i.e., by means of a web server, such as *nginx* or *apache*).

The preparation of the Dockerfile with the appropriate instructions is
left to the reader, which shall leverage the concepts learned during the
previous laboratories.

> [!TIP]
> In order to reduce the resulting image size, it is good practice to leverage the **multi-stage build** approach.
> In a nutshell, a first container including all the necessary software is in charge of all the preparatory steps (e.g., build the website), while the resulting artifacts are copied to a second image, including only the applications required at runtime (e.g., the webserver).

Once the Dockerfile is complete, it is possible to build the image and
verify it works correctly by executing it as a local container. You
should be able to navigate the resulting website.

If the website works properly, the last step involves **pushing** the
image to a public repository, such as Docker Hub or Harbor, so that it
can be lated retrieved by Kubernetes.

> [!NOTE]
> Pay attention to tag the image correctly: in case it is pushed to Docker Hub, it shall be prepended with your username, while for Harbor you need to specify the entire registry url, the `cloud-sandbox` repository and prefix the image name with your PoliTO ID.
> Do always specify a version tag for your images, to ensure the correct version is used in production.