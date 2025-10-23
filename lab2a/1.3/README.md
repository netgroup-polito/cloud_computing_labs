<table style="width:100%">
  <tr>
    <td align="left"><a href="../1.2/README.md">⬅️ Previous</a></td>
    <td align="right"><a href="../1.4/README.md">Next ➡️</a></td>
  </tr>
</table>

# 3. Linux Namespaces

Quoting Wikipedia, "The **Linux namespaces** are a feature of the Linux kernel that partitions kernel resources such that one set of processes sees one set of resources while another set of processes sees a different set of resources."

In other words, namespaces enable the creation of distinct **virtual environments** with respect to a given type of kernel resource, so that different namespaces have a completely independent view of it. For instance, two **network** namespaces are associated with independent networking stacks, virtual interfaces, IP addresses, routing tables, and so on.

As of today, the Linux kernel provides eight types of namespaces, which constitute one of the main **building blocks** of lightweight virtualization.

In the following sections, you will perform some simple experiments with three of them:

* **Process ID** (`PID`): Used to create **different process trees**. A process in a new PID namespace has its own set of PIDs, starting from 1, and only sees other processes within the same namespace.
* **Mount** (`MNT`): Used to create a completely **new view of the filesystem** with the desired structure. Changes to the filesystem (like mounting/unmounting) are local to the namespace.
* **Network** (`NET`): Used to allow two processes to perceive a totally **different network setup**, including separate interfaces, IP addresses, routing tables, and firewall rules.