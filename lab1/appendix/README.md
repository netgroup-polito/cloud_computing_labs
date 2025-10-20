<table style="width:100%">
  <tr>
    <td align="left"><a href="../1.1/README.md">⬅️ Back to Sec 1.1</a></td>
    <td align="left"><a href="../README.md">⬅️ Back to Home</a></td>
  </tr>
</table>

# Appendix: setting up KVM 

If you are using the VM provided with the lab, you can skip this section.
In fact, the VM already contains all the tools needed for the lab assignment.
This section will be useful if you have to rebuild a VM with the required environment from scratch.

KVM is a Linux kernel module that turns the standard Linux operating system into an hypervisor, hence introducing kernel support for VMs.

In order to install KVM on a recent Ubuntu distribution, follow the detailed instructions [here](http://help.ubuntu.com/community/KVM/Installation).
We strongly suggest to install also the `virt-manager` GUI, which is marked as optional in the above guide.