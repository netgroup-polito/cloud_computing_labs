<table style="width:100%">
  <tr>
    <td align="left"><a href="../2.5a/README.md">⬅️ Previous</a></td>
    <td align="right"><a href="../README.md">End ✅</a></td>
  </tr>
</table>

# 6. Linux cgroups: Defining Maximum CPU Consumption

**Linux cgroups** (control groups) represent the second building block necessary to implement lightweight virtualization.
Essentially, cgroups are a Linux kernel feature providing fine-grained control to limit, account, isolate, or deny resource usage to a process or a group of processes.

In this section, we use cgroups to **limit the amount of CPU a given process can use**.
You will need the tools provided by the `cgroup-tools` package.

> [!NOTE] 
> Linux includes cgroups in all recent kernels; however, the userspace tools required to control them are not installed by default.
> If you encounter issues, refer to the installation instructions in the relevant section.

## 6.1. Simulating a CPU-Intensive Task

First, create a simple **bash script** that simulates a CPU-intensive task through an infinite loop:

```bash
# Create a specific folder for cgroups experiments
mkdir --parents docker-lab/cgroups

# Create a script that endlessly consumes your CPU
cat <<'EOF' > docker-lab/cgroups/infinite.sh
#!/bin/bash
i=0
while true; do
  i=$((i+1));
done
EOF

# Change the permissions of the above file and make it executable
chmod +x docker-lab/cgroups/infinite.sh
```

## 6.2. Running Multiple Processes Without cgroups

To check the default behavior when cgroups are not configured, execute the `infinite.sh` script three times, passing a different parameter to each one to make them distinguishable during monitoring.

In this example, the `taskset` utility is used to **force all processes onto the same CPU core** (to ensure comparable results):

```bash
taskset 1 docker-lab/cgroups/infinite.sh N1 &
taskset 1 docker-lab/cgroups/infinite.sh N2 &
taskset 1 docker-lab/cgroups/infinite.sh N3 &
```

To monitor the CPU usage of these processes, use `htop`, or run the following command for a more compact view:

```bash
htop --pid=$(pgrep --full --delimiter=, "infinite.sh")
```

As expected, the three `infinite.sh` processes will quickly saturate the first CPU core, each consuming roughly **33% of the CPU**.

Before proceeding, stop the running processes:

```bash
pkill --full "infinite.sh"
```

## 6.3. Running Multiple Processes With cgroups

Now let’s use **cgroups** to control CPU usage more precisely.

### 6.3.1. Create the cgroups

Create two new ad-hoc CPU cgroups, named `throttle25` and `throttle75`:

```bash
sudo cgcreate -a "$USER" -t "$USER" -g cpu:/throttle25
sudo cgcreate -a "$USER" -t "$USER" -g cpu:/throttle75
```

### 6.3.2. Configure CPU Shares

Assign different CPU shares to each group:

```bash
cgset -r cpu.shares=512 throttle25
cgset -r cpu.shares=1536 throttle75
```

Here:

* `throttle25` receives 512 shares (≈25% CPU time)
* `throttle75` receives 1536 shares (≈75% CPU time)

These percentages are *relative* to each other. The default value of `cpu.shares` (1024) has no effect since we set explicit values.

> [!NOTE]
> Setting `cpu.shares` does **not** guarantee throttling to that percentage.
> The kernel scheduler enforces these values **only when CPU contention exists** (i.e., multiple processes compete for the same core).

### 6.3.3 Run Processes in cgroups

Now execute the scripts again using `cgexec` to assign each process to a specific cgroup:

```bash
cgexec -g cpu:throttle25 taskset 1 docker-lab/cgroups/infinite.sh N1 &
cgexec -g cpu:throttle25 taskset 1 docker-lab/cgroups/infinite.sh N2 &
cgexec -g cpu:throttle75 taskset 1 docker-lab/cgroups/infinite.sh N3 &
```

Monitor the processes again with:

```bash
htop --pid=$(pgrep --full --delimiter=, "infinite.sh")
```

You should now observe **uneven CPU distribution**:

* The process in `throttle75` uses ~75% of the CPU core.
* The other two processes (in `throttle25`) share the remaining ~25%.

## 6.4. Setting Hard CPU Limits

In addition to relative shares, you can configure **hard CPU usage limits** using the following parameters:

* `cpu.cfs_quota_us`
* `cpu.cfs_period_us`

For example, to limit a cgroup to **50% CPU usage**:

```bash
cgset -r cpu.cfs_quota_us=50000 my_cgroup
cgset -r cpu.cfs_period_us=100000 my_cgroup
```

Experiment with these parameters to understand their effects.

---

## 6.5. Cleaning Up

Before moving to the next part of the lab, stop all running processes and delete the created cgroups:

```bash
pkill --full "infinite.sh"

sudo cgdelete -g cpu:/throttle25
sudo cgdelete -g cpu:/throttle75
```

