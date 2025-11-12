# Setting up OpenStack and creating VM-based services 

## Introduction

The goal of this exercise is to practice with an **VM-oriented cloud management framework such as OpenStack**, namely installing this platform, and creating and playing with VM-based services on it.

To show the successful completion of the lab, **students will be requested to upload a screenshot of the service they created**, according to the assignment available on the course website (Moodle).

## Physical setup

Since OpenStack requires non negliglible skills to be installed and careful planning, in this lab we suggest to install *DevStack*, a developer-oriented version of OpenStack. DevStack is a full-functioning version of OpenStack, with all its main features, but packaged in a way to be easily installed with a few commands, and to be self-contained on a single host (e.g., a VM with minimal requirements; we suggest 4 CPU cores, 12GB RAM, 20GB disk space).

The setup involves two hosts:

-   **A first VM, hosting OpenStack**. This VM, running on CrownLabs, is based on Ubuntu 22.04 LTS and named `Ubuntu 22.04 - CloudVM`.
    This VM is based on a cloud image, hence it includes the most common tools required for a production-grade server-based environment.
    In Crownlabs, it has been configured to be *persistent* and console-only (hence, no GUI).
    This VM will host your DevStack environment.

-   **A second VM, hosting a management workstation**. This VM, running on CrownLabs, provides GUI-based access to the previous one, such as the `Cloud Computing: Client VM` instance.

**Note**: although the second VM seems avoidable, potentially being replaced by an SSH connection from the student's computer to the DevStack VM, we suggest to use it because of the necessity to use a web browser to interact with the DevStack machine. In fact, the interaction with OpenStack is web-based and may rely also on non-standard TCP ports when the console of running VMs is opened, which may be difficult to handle through an SSH connection.

# Installing OpenStack

The installation of OpenStack is rather simple, by opening a console-based connection to the DevStack VM and following the instructions on the DevStack website:

<https://docs.openstack.org/devstack/latest/>

**Note**: the installation process usually takes between 30 and 60 minutes, depending on the characteristics of your VM.

## Install on a custom VM (not on CrownLabs)

For completeness, we report here some hints in case you would prefer to install DevStack on your own computer, e.g., on a VM running locally:

-   **Use a cloud image**. Do not start from a vanilla Linux OS; choose a cloud image available here, preferably a recent LTS version: <https://cloud-images.ubuntu.com/>

-   **Resize the image disk**. Cloud images have usually rather small disk size.
    Before running the cloud image, resize its disk to be at least 20GB.
    For example, if you are using KVM, you can type the following command:
    
        qemu-img resize ubuntu-22.04-server-cloudimg-amd64.img +15G

    This increases the disk size of the `ubuntu-22.04-server-cloudimg-amd64.img` image of additional 15GB.

-   **Install git**. DevStack requires `git` to be installed. If not already present on the VM image, you can install it (on Ubuntu) with `apt install -y git`.

-   **Assign enough resources to the VM**. We suggest at least 2 CPU cores, 6 GB RAM, but better if you have more, e.g., to avoid the installation stopping without any meaningful message because of lack of memory or disk full.

**Note**: in case you are not able to clone the DevStack git repository because of a failure in the server certificate verification, which is one of the first commands in the DevStack setup, re-start the `git clone` command with the `-c http.sslVerify=false` flag (e.g., `git -c http.sslVerify=false clone <path>`).

## Configuring and playing with OpenStack

Once the setup is completed, you can connect to your OpenStack instance through a web browser executed on the second VM.
However, you have to pay attention to the following issues.

-   **IP address of the OpenStack VM**. When the DevStack script finishes, one of the messages shown on screen looks like the following:

        This is your host IP address: 172.16.183.136

    To connect to the OpenStack VM from the management VM, **you have to
    use this IP address** (i.e., the one shown at the end of the
    installation) and not the one reported in the CrownLabs dashboard,
    which usually is in the `10.0.0.0/8` range. In case you missed the
    above message, you can connect via SSH to the OpenStack VM and type
    the command `ip addr`, which shows the real IP address of the VM.
    Take note of that address, and use it to connect from the second VM.

-   **Enable DHCP on the public network**. OpenStack comes with two
    logical networks already configured, which are visible by entering
    in the OpenStack dashboard with the admin user (and the password you
    set during the installation process), then selecting "Project",
    "Network", and "Network Topology". The *public* network is the one
    that allows VMs running in OpenStack to connect to the Internet but,
    by default, the DHCP service is disabled on that network. Use the
    OpenStack dashboard to edit the properties of that network and
    enable the DHCP service *before* creating any VM.

-   **Cannot stop and relaunch OpenStack**. The DevStack installation of
    OpenStack works in a \"fire and forget\" fashion. If you reboot your
    machine, the installaiton will no longer work and you have to
    reinstall it from scratch, starting from a fresh *operating system*
    image. If you want to install OpenStack again on the same VM, you
    have to type the above commands:

        # Uninstall OpenStack
        ./unstack.sh
        # Clean all. Note: the source files (i.e., the result of the git clone command) are
        # not deleted, hence you may not need to clone the repository again upon reinstall.
        ./clean.sh
        sudo reboot
        # Restart the installation script from scratch
        ./stack.sh

The OpenStack install is now ready for you to play and to deliver what the assignment (on Moodle) asks you for.
