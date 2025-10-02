<table style="width:100%">
  <tr>
    <td align="left"><a href="../1.1/README.md">⬅️ Previous</a></td>
    <td align="right"><a href="../1.3/README.md">Next ➡️</a></td>
  </tr>
</table>

# 2. Prepare the lab environment

## 2.1. Client VM (a.k.a. management host)
The **management host** is used to control the remote hypervisor running on the remote server.

This machine will use SSH to connect to the remote server, and it will also exploit the GUI-based Virtual Machine Manager (`virt-manager`), which makes interactions with the remote hypervisor easier.

While the KVM hypervisor can be controlled using command-line tools (for example, `virsh` to create VMs), this is not that easy at the beginning (remember that servers usually do not have any graphical interface, hence all interactions must be carried out using console-mode operations).  
Instead, `virt-manager` provides a graphical interface (so it cannot be executed directly on the server) that can control a remote hypervisor by issuing the proper commands through an SSH connection. Hence, `virt-manager` facilitates interaction with a remote hypervisor that works in console-mode only (no GUI).

---

## 2.2. Server VM (a.k.a. student's server)
The server comes with all the required tools preinstalled, so nothing has to be done to prepare this environment.

### 2.2.1. Access to the server
First, get the IP address of the server (the KVM virtual machine) from the CrownLabs dashboard.  
Access the server **from your management host** with:

```bash
ssh crownlabs@IP_of_KVM_VM
```

If you are successfully authenticated, configure a proper SSH key pair. Key pairs rely on asymmetric authentication and are normally preferable for SSH. Key-based authentication allows you to disable password authentication (reducing the risk of dictionary attacks) and can be non-interactive — enabling automated, agentless configuration workflows (used by tools such as Ansible).

Once you have accessed the server via SSH, make sure the crownlabs account has permission to interact with the libvirt daemon by adding it to the corresponding group:

```bash
sudo adduser crownlabs libvirt
```