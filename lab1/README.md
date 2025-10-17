# Lab 1 - Computing Virtualization with KVM

This lab aims at practicing with the creation and configuration of virtual machines (VMs) in a computing virtualization framework, namely **Linux KVM** ([https://www.linux-kvm.org/](https://www.linux-kvm.org/)).  
This is the default hypervisor in Linux, oriented to production-grade virtualization, available only in the above OS, targeting preferably console-based guests such as Linux servers, although GUI-enabled Guest OSes can be supported as well.

As a more complex scenario, this lab additionally simulates a configuration in which two hosts that belong to different LANs can exchange traffic through a router, connected to the same switched network of the previous hosts.

**EXTRA**: In addition to this text, we provide several videos which discuss some parts of the topics of this lab. You can find them on [YouTube](https://www.youtube.com/playlist?list=PLTAfidx4guQImT5beuAs4YAhIzuBBoEHk).

## Table of contents 

[Chapter 1 - Practicing with virtual machines](./1.1/README.md)

[Chapter 2 - Prepare the lab environment](1.2/README.md)

[Chapter 3 - Configure proper storage folders on the server](1.3/README.md)

[Chapter 4 - Download the Ubuntu Cloud image disk](1.4/README.md)

[Chapter 5 - Customize the disk image with your personal data](1.5/README.md)

[Chapter 6 - Create a new VM](1.6/README.md)

[Chapter 7 - Inspect the Network Configuration](1.7/README.md)

[Chapter 8 - Exchange Traffic Between Two VMs Through a Third VM Acting as Router](1.8/README.md)

[Appendix: Setup KVM](appendix/README)
