<table style="width:100%">
  <tr>
    <td align="left"><a href="../README.md">⬅️ Previous</a></td>
    <td align="right"><a href="../1.2/README.md">Next ➡️</a></td>
  </tr>
</table>

# 2. Traditional Filesystem Isolation

## 2.1. Introduction

This part of the laboratory aims to provide a practical introduction to the main functionalities provided by the **Linux kernel** to implement strong **process isolation**.

Although usually not exploited directly by final users nor by general-purpose programmers due to their complexity, ***namespaces*** and ***cgroups*** represent the main building blocks upon which **lightweight virtualization** is constructed.

Hence, the main goal of the following is to gain better confidence with some of the technologies exploited under the hood by ***Linux Containers*** and ***Docker*** while performing some simple tasks.

## 2.2. Chroot

This section provides an introduction to one of the filesystem isolation facilities provided by the Linux Kernel: ***chroot***.

Its purpose is to change the apparent **root directory** (i.e., `/`) for one process and its children. Thus, a program executed in such a modified environment cannot see and access files outside the designated directory tree, effectively creating a sort of ***jail***.

> [!NOTE] 
> Please note that it is fairly easy to escape the jail if you can get `root` privileges within the jail itself. See, for instance, [https://warsang.ovh/prison-break-chroot/](https://warsang.ovh/prison-break-chroot/) and its references for more information.

Generally speaking, a `chroot` environment can be exploited to create and host a separate virtualized copy of the software system, where applications can be executed for testing purposes, along with all the expected dependencies, without risking to modify and damage the hosting system. Nonetheless, one of the most known usages of `chroot` is probably associated with the **recovery of unbootable systems**. In this situation, the usually adopted solution consists in booting a live distribution and then using `chroot` to move back to the corrupted environment, to e.g., reinstall the bootloader and perform any recovery task that may be required.

In the following, you will be guided along the execution of a `bash` shell within a `chroot` environment, to verify the provided level of isolation.

### 2.2.1. Setting up the `chroot` Environment

To begin, let's create the directory that will be mounted as `/` in the `chroot` environment:

```bash
mkdir --parents docker-lab/chroot
```

Then, copy the `bash` executable into the `docker-lab/chroot/bin` directory:

```bash
mkdir --parents docker-lab/chroot/bin
cp --verbose /bin/bash docker-lab/chroot/bin
```

Before executing the `chroot` command, you need to copy to the chroot system all the libraries required by `bash`, which can be listed by typing `ldd /bin/bash`.

In this respect, please, note that:

  * `linux-vdso.so` is a special library that is directly provided by the kernel. Hence, it is not associated with a file name and you do not need to copy it in the chroot environment.
  * All the required libraries need to be copied in the `docker-lab/chroot` folder, **preserving the original folder structure** (e.g., if a library is under `/lib64` in the root filesystem, it must be present in the same path of the *chrooted* environment).

In case of problems, you can leverage the following command that copies all the required files and preserves the original folder structure:

```bash
for LIB in $(ldd /bin/bash | grep --only-matching --perl-regexp '/.*(?= \(0x)'); do \
    DEST="docker-lab/chroot$LIB"; \
    mkdir --parent "$(dirname $DEST)"; \
    cp --verbose "$LIB" $DEST; done
```

Finally, observe the content of the `chroot` directory by typing `tree docker-lab/chroot` (or, if you prefer, `ls -l --recursive docker-lab/chroot`).

### 2.2.2. Entering the `chroot` Jail

Now, you are ready to create the `chroot` environment with the following command:

```bash
sudo chroot --userspec="$USER" docker-lab/chroot /bin/bash
```

where:

  * we enter the `chroot` environment with the current user as root;
  * the `chroot` environment is created starting from `docker-lab/chroot` folder;
  * once we enter, we execute the `bash` shell. Please note also that the `/bin/bash` path is relative to the `chroot` system, thus the command will invoke `bash` from `docker-lab/chroot/bin` and not from the `/bin` folder of the host system.

> [!NOTE] 
> in case you experience an error like `chroot: failed to run command '/bin/bash': No such file or directory` it may be that the libraries are not all in the proper place, while the executable may be ok.

If the `chroot` command succeeds, your `bash` prompt should have changed to `bash-x.y#`, with `x.y` being its version number.

> [!NOTE] 
> Prompt `bash-x.y#` is the default one used by `bash` in case no customized directives for the prompt are available in the user's home folder.

### 2.2.3. Experimenting in the Jail

Now, it is possible to perform some simple experiments from within the `chroot` jail. However, please notice that only `bash` and its built-in commands can be invoked, while any other command (e.g., `ls`) will fail with a `command not found` error, since the corresponding executable is not present within the jail.

Now, let's explore the root filesystem from the `chroot` environment, remembering that the `ls` command is not available within the `chroot` environment, hence it needs to be emulated as follows:

```bash
# shopt is a shell builtin command to set and unset (remove) various
#    bash shell options. Here we set globstar, which enables the
#    double asterisk, which expands to any number of directories.
shopt -s globstar
for PATH in /**/*; do echo "$PATH"; done
```

As you can see looking at the listed elements, only the files and directories contained in the `chroot` directory can be accessed from within the `chroot` environment.

Now, let's create a new file in the (`chroot`) root folder:

```bash
echo "This is a test file created within the chroot environment" > testch.txt
```

This file would be visible under `/testch.txt` from within the `chroot` environment, while under `docker-lab/chroot/testch.txt` when opening a new terminal operating outside the `chroot` environment.

As evident, the `chroot` user cannot escape from its environment, but the base system has the complete visibility over the entire filesystem, including the `chroot` folders.

Before moving on to the next section of the laboratory, remember to exit from the `chroot` environment (i.e., type `exit` to quit bash).