<table style="width:100%">
  <tr>
    <td align="left"><a href="../2.3a/README.md">⬅️ Previous</a></td>
    <td align="right"><a href="../2.5a/README.md">Next ➡️</a></td>
  </tr>
</table>


# 5. Mount (MNT) Namespaces

*Slightly adapting the introduction provided by [LWN.net](https://lwn.net/Articles/689856/):*

**Mount namespaces (`MNT`)** are a flexible tool that allows isolating the list of mount points seen by the processes in a namespace.
In other words, each `MNT` namespace has its own list of mount points, meaning that processes in different namespaces see and are able to manipulate different views of the single directory hierarchy.

When the system boots, there is a single mount namespace — the so-called **initial namespace**.
Then, new `MNT` namespaces can be created, with a copy of the mount point list replicated from the namespace of their creator.
Finally, mount points can be independently added and removed in each namespace.
Changes to the mount point list are by default visible only to processes in the mount namespace where the process resides: they are **not visible** in other mount namespaces.

## 5.1. Running a Shell in a New Mount Namespace

In the following, you will be required to run a `bash` shell in a new `MNT` namespace.
Then, you will modify some of the mount points and verify that they are independent from the ones present in the initial namespace.

To begin, use the `unshare` utility with the `--mount` flag to create a new `MNT` namespace:

```bash
sudo unshare --mount /bin/bash
```

## 5.2. Observing the Mount Points

Differently from what happened when playing with `chroot`, the newly created namespace contains **the same mount points** as the original one.
Hence, you can access the very same files and programs available in your original system — in fact, `bash` was started from its original location (`/bin/bash`).

Now, let's create a new mount point:

```bash
sudo mkdir --parents /mnt
sudo mount --bind docker-lab/chroot/ /mnt/
```

Here, the `mount` command **remounts** (notice the `--bind` flag) the `docker-lab/chroot/` directory and all its subdirectories — the ones you created in the previous section — with target `/mnt`.
Hence, if you now type:

```bash
tree /mnt
```

you will see the directory structure you created previously.

## 5.3. Creating a Test File

Now create a new file in the new mount point, named `/mnt/testns.txt`:

```bash
echo "This is a test file created within the new mount namespace" > /mnt/testns.txt
```

You can verify that it is reflected back to the `docker-lab/chroot/` folder.

## 5.4. Verifying Namespace Isolation

To verify the isolation between different `MNT` namespaces, open a **new terminal** (thus, back to the original `MNT` namespace) and follow the steps below:

* Show the content of the `/mnt` directory. Is there any file?
* Type `tree docker-lab/chroot/`. Can you see the `testns.txt` you created previously? Why?
* List, in both terminals, all the mounted filesystems by typing:

  ```bash
  findmnt
  ```

  Is there any difference between the two outputs?
  *Tip: try to identify the mount point you created previously (having target `/mnt`).*

## 5.5. Typical Use of MNT Namespaces

In a typical scenario, `MNT` namespaces would be used **in combination with `chroot`** to “customize” the filesystem view seen by a process.

Simplifying a bit the different steps:

1. A new `MNT` namespace would be created and entered to avoid cluttering the original one with additional mount points.
2. One would perform all the `mount` and `umount` operations required to set up the environment.
3. Finally, the `chroot` command would be executed to switch the process root.

## 5.6. Exiting the Namespace

Before moving on to the next section of the laboratory, exit from the `MNT` namespace:

```bash
exit
```
