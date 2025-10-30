<table style="width:100%">
  <tr>
    <td align="left"><a href="../README.md">⬅️ Previous</a></td>
    <td align="right"><a href="../1.2/README.md">Next ➡️</a></td>
  </tr>
</table>

# 1. Physical Setup

![Physical setup: how to connect to your virtualized resources.](images/Lab-physical-setup.png)
*Figure: Physical setup – how to connect to your virtualized resources.*

This laboratory requires a physical setup as shown in the figure above.
Each student is provided with **two virtual machines (VMs)** running in the **POLITO datacenter**, accessible through the **[CrownLabs dashboard](https://crownlabs.polito.it)**. These VMs represent a **client** and a **server**.

## 1.1. Client VM

A Linux (or equivalent) client should be provisioned using the **“Cloud Client”** template.
This machine acts as the **management host**, which you will use to configure the (remote) server environment for this lab — specifically, the **“Cloud Computing: Docker”** laboratory template.

## 1.2. Server VM

On the **server** side, the lab operates with a **single VM**.
All exercises and Docker-related experiments are performed on this server machine.

## 1.3. Alternative Access (SSH)

If you have installed an **SSH key** on your local machine, it is possible to connect **directly** to the Server VM, bypassing the use of the Client VM.
Please follow the instructions provided on the **CrownLabs website** (see the *Resources* section) for details on how to use this method.