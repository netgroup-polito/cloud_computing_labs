<table style="width:100%">
  <tr>
    <td align="left"><a href="../1.4/README.md">⬅️ Previous</a></td>
    <td align="right"><a href="../1.6/README.md">Next ➡️</a></td>
  </tr>
</table>

# 5. Customize the disk image with your personal data

Cloud images are operating system templates and every instance starts out as an identical clone of every other instance.  
Before running the instance, images may need to be customized (e.g., adding software packages, modifying configuration parameters such as network, or creating proper credentials for login).

> [!NOTE]
> Security: the default user account on the Ubuntu image (`ubuntu`) has login disabled from both console and SSH.

In this lab, you simply need to enable a user account to be able to login.  
Two possible procedures are presented:

1. **Using `cloud-init` (preferred)**
2. **Using `virt-customize`**

---

## 5.1. Cloud-init (preferred)

[`cloud-init`](https://cloud-init.io/) ([documentation](https://cloudinit.readthedocs.io/en/latest/)) is one of the most widely used tools for passing initialization parameters to a VM at boot time.  

It relies on two text files that are read either from a predefined network location or (in this lab) from a disk drive mounted to the VM at boot as a CD-ROM device.

This procedure adds a new user to the VM and pushes the user's SSH public key in the VM itself, so that you will be able to login using your private SSH keys.
In addition, it also adds a password for the default user (`ubuntu`), allowing that user to log-in only through the console.
This procedure may be convenient in case, for any reason, you lose the connectivity with your server, hence logging in via console is the only way to recover your VM.
In any case, be careful that the access via username/password is considered less secure compared to the use of a public/private key.


1. If you don’t already have them, create a pair of public/private SSH keys with `ssh-keygen`.  
   A good reference is [this tutorial](https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys-on-ubuntu-1604) (only Step 1 is needed).

2. Create a new text file named `user-data` in your home directory.  
   If unfamiliar with Linux console editors, `nano` is a simple choice.

3. Write the following content in the file, replacing the `ssh-rsa ...` line with the full text of your own public key (usually found in `~/.ssh/id_rsa.pub`):

   ```yaml
   #cloud-config
   users:
     - default
     - name: ubuntu
       ssh_authorized_keys:
         - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCjYqAlhqnt7aOSv4gdSMSKWE09YTP2Q2Wv5Uygnx...
       sudo: ALL=(ALL) NOPASSWD:ALL
       groups: sudo
       shell: /bin/bash
   ```

   > [!WARNING]
   > YAML indentation is critical. Do not use tabs and ensure consistent spaces.

4. Create another text file named `meta-data` with the following content:

   ```yaml
   local-hostname: test-vm1
   ```

5. Use the tool `cloud-localds` to create a configuration disk (called a *seed* in `cloud-init` terminology):

   ```bash
   cloud-localds seed-init.iso user-data meta-data
   ```

   This produces a file `seed-init.iso` containing the two files:

   ```
   root/
   ├── user-data
   └── meta-data
   ```

6. Copy the ISO into your `~/kvm-iso` folder:

   ```bash
   cp seed-init.iso ~/kvm-iso/.
   ```

---

### 5.1.1. Cloud-init and persistent disks

This lab uses **KVM** as virtualization framework. Changes applied by `cloud-init` are written directly to the OS image and persist across runs.  

This differs from systems like **OpenStack**, where the OS image remains immutable, and `cloud-init` must always be re-applied.

Observations:

1. In KVM, the `cloud-init` disk may be required only at the first boot, since changes persist in the image. The disk can be removed in later runs without effect.  
   In OpenStack, the disk must always be present.

2. Replacing an already applied `cloud-init` disk with a different one may not work if it attempts to modify existing users.  
   - Adding a new user/key usually works.  
   - To change existing credentials reliably, it is safer to reinitialize the OS image (restore the original Ubuntu cloud image).

---

## 5.2. Virt-customize (not needed in lab)

> **IMPORTANT**: This section is for reference only. **Do not perform these steps in the lab.**

[`virt-customize`](http://libguestfs.org/virt-customize.1.html) (from the optional `libvirt` package `libguestfs-tools`) can modify a disk image directly.  
Unlike `cloud-init` (which changes the OS at boot), `virt-customize` permanently alters the image filesystem.

Example: set the root password to `coolpwd` in `focal-server-cloudimg-amd64.img`:

```bash
virt-customize -a focal-server-cloudimg-amd64.img     --root-password password:coolpwd
```

Now you can login from console as `root` with the password you set.

**Notes**:
1. The VM image content is permanently changed.
2. `virt-customize` can also add new users, but this requires scripting. Since the lab uses `cloud-init`, more advanced `virt-customize` usage is left to the student.

---

## 5.3. Resize the VM disk (not needed in lab)

If you need more disk space in a vanilla image (e.g., to install large packages), you can resize it with `qemu-img`:

```bash
qemu-img resize focal-server-cloudimg-amd64.img +15G
```
