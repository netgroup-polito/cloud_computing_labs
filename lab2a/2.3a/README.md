<table style="width:100%">
  <tr>
    <td align="left"><a href="../2.2a/README.md">⬅️ Previous</a></td>
    <td align="right"><a href="../2.4a/README.md">Next ➡️</a></td>
  </tr>
</table>

# 4. Process ID Namespaces

As a first example of namespaces, let's experiment with the creation of a **Process ID (`PID`) namespace**.

## 4.1. Concept

In Linux, processes originate a single process tree, with the root being the **`init` process** (PID=1), which is in charge of special tasks (e.g., all orphaned processes are attached to it).

**PID namespaces** isolate the process ID number space. This means:

1.  Processes in different PID namespaces can have the **same PID** (e.g., each can have its own PID=1).
2.  A process can create a new process tree with its own **PID=1** process, which receives special treatment similar to the normal `init` process.
3.  PID namespaces are **nested**. A process has a PID for each namespace from its current one up to the initial (host) namespace.
4.  The initial (host) PID namespace is the only one able to **see all processes** across all nested namespaces.

## 4.2. Experiment: Creating a New PID Namespace

You will use the `unshare` utility, which allows you to run a program with some namespaces unshared from its parent.

Run the following command:

```bash
sudo unshare --pid --mount-proc --fork /bin/bash
```

Where the flags mean:

  * `--pid`: Specifies the creation of a **new `PID` namespace**.
  * `--mount-proc`: Mounts a new `proc` filesystem at `/proc` to ensure the executed process gets **PID=1** (this also creates a new `MNT` namespace under the hood).
  * `--fork`: Forces `unshare` to **fork** before running the command instead of running it directly, again to help ensure the process gets **PID=1**.
  * `/bin/bash`: The command to be executed (the `bash` shell).

> [!NOTE]
> In case you get the error *bash: /root/.bashrc: Permission denied*, you can safely ignore it (or append the `-norc` flag to the `bash` invocation to remove the message).

## 4.3. Observation and Verification

Inside the New Namespace issue the `htop` command in the newly created namespace:

  * You will observe that the `/bin/bash` process got assigned **PID=1**.
  * `htop` is displaying **only the processes** running in the current namespace (the new `bash` at PID 1, and `htop` itself).

To go back on the Host (Initial Namespace):

1.  Open a **new terminal** (you are now back in the initial/host `PID` namespace).
2.  Issue the `ps -af` command.

**Question:** Can you identify the `/bin/bash` (PID 1 in the new namespace) and `htop` processes running in the newly created namespace? If yes, what are the PIDs associated with them **on the host system**?

> [!TIP]
> They will appear as children of the `unshare` command you ran.

-----

Before moving on to the next section of the laboratory, remember to exit from the `PID` namespace (i.e., type `exit` in the first terminal to quit the nested `bash` shell).