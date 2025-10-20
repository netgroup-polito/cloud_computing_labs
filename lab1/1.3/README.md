<table style="width:100%">
  <tr>
    <td align="left"><a href="../1.2/README.md">⬅️ Previous</a></td>
    <td align="right"><a href="../1.4/README.md">Next ➡️</a></td>
  </tr>
</table>

# 3. Configure proper storage folders on the server
> [!IMPORTANT]
> Unless explicitly specified, all the instructions below apply to the **server** (i.e., `Cloud Computing: KVM` template VM in Crownlabs).  
> This lab assumes that the student is already logged into the server.

---

In order to create new VMs, we may need to store:  
- A set of ISO images (OS install disks or configuration disks used by `cloud-init`)  
- A set of VM images (the actual installed VM images) on the KVM server.  

To avoid polluting your system, you should create two new directories:  
- One to keep your ISO (install) disks  
- One for your storage pools (installed VM images)  

The default storage pool in KVM is `/var/lib/libvirt/images`.  
Instead, we suggest creating two distinct folders under your home directory: `~/kvm-iso` and `~/kvm-pool`, as follows:

```text
/home/crownlabs/
         |
         +----/kvm-iso
         |
         +----/kvm-pool
