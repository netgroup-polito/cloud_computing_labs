# Introduction

The goal of this laboratory is to guide the reader through the main technologies behind **lightweight virtualization** and some of its possible applications.

Over the years, the necessity for **strong process isolation** has been widely recognized. This isolation, for example, allows the execution of arbitrary code on a third-party server and prevents a possible intrusion in one service from compromising the entire machine it's running on.

Nonetheless, classical full-blown virtualization is often considered undesirable due to the associated overheads, especially in terms of memory requirements and operating system start-up time.

To solve these issues, new functionalities have been gradually introduced in the Linux kernel over the last two decades. These allow multiple programs to perceive a different and isolated view of a part of the resources available in a system, as well as to limit and account for their usage.

You will experiment with some of these techniques, namely **namespaces** and **cgroups**, in the relevant chapter/section.

**EXTRA**: In addition to this text, we provide some videos that present some of the topics of this Lab. You can find them at the following on [Youtube](https://www.youtube.com/playlist?list=PLTAfidx4guQImT5beuAs4YAhIzuBBoEHk).

## Table of contents 

[Chapter 1 - Physical setup](./1.1/README.md)

[Chapter 2 - Traditional Filesystem Isolation](./1.2/README.md)

[Chapter 3 - Linux Namespaces](./1.3/README.md)

[Chapter 4 - Process ID Namespaces](./1.4/README.md)

[Chapter 5 - Mount (MNT) Namespaces](./1.5/README.md)

[Chapter 6 - Network Namespaces](./1.6/README.md)
