# eBPF Observability on RHEL with BCC Tools

## 1. What you will learn

In this lab, you will use **eBPF-based BCC tools** to observe what happens inside a Linux system while `dnf update` runs.

You will monitor four main areas:

| Area | Tool | What it helps you see |
|---|---|---|
| DNS lookups | `gethostlatency` | Which hostnames applications resolve, and how long lookups take |
| TCP connections | `tcplife` | Which processes open TCP connections, where they connect, and how much data they transfer |
| File activity | `filetop` | Which processes read and write files, and how much I/O they perform |
| Slow XFS operations | `xfsslower` | XFS file operations that take longer than a defined threshold |
| Memory page cache | `cachestat` | Cache hits and misses while applications access files |

**Simple example:** When you run `dnf update`, the system may:

1. Resolve a repository hostname, such as `cdn.redhat.com`.
2. Open an HTTPS connection to download repository metadata and packages.
3. Read package files and write updated files to disk.
4. Access data that may or may not already be in memory cache.

The tools let you observe these activities while they happen.

> **Important:** Run each monitoring command in its own terminal tab and leave it running. Run `dnf update -y` in a separate terminal tab.

---

## 2. Prepare the environment

### Step 1: Install `bpftool`

In the **dnf** terminal, install the utility:

```bash
dnf install -y bpftool
```

`bpftool` is a command-line utility for inspecting and managing eBPF objects, including programs loaded into the kernel.

### Step 2: Check the eBPF programs already loaded

Run:

```bash
bpftool prog list
```

To show only entries containing `cgroup` and the following three lines, run:

```bash
bpftool prog list | grep -A 3 cgroup
```

If no matching output appears, there may be no entries matching `cgroup`. This does **not** necessarily mean that no eBPF programs of any type are loaded. Check the complete output from `bpftool prog list` to see all listed programs.

You may see programs associated with the system's cgroup implementation. These are existing kernel eBPF programs; the BCC tools you start in the next steps will add their own observability programs.

---

## 3. Start the BCC monitoring tools

Open the corresponding terminal tab for each tool. Run the command shown below and keep it running.

### 3.1 Monitor hostname lookup latency: `gethostlatency`

In the **gethostlatency** terminal, run:

```bash
/usr/share/bcc/tools/gethostlatency
```

This tool observes calls such as `getaddrinfo()` and `gethostbyname()` and reports how long hostname lookups take.

**Why is this useful?**

When an application connects to a hostname, it may first need to resolve that name to an IP address.

For example, while `dnf` contacts a repository, you may see lookups for names such as:

- `cdn.redhat.com`
- `subscription.rhsm.redhat.com`
- `localhost`

This can help identify hostname lookups and investigate DNS-related delays.

**Example interpretation (illustrative):**

If the output shows `cdn.redhat.com` with a lookup latency of `25 ms`, the lookup took approximately 25 milliseconds. The exact columns and output format depend on the installed BCC version.

### 3.2 Monitor TCP connections: `tcplife`

In the **tcplife** terminal, run:

```bash
/usr/share/bcc/tools/tcplife
```

The tool reports information about TCP connections when they close. You should see a header similar to the following:

```text
PID   COMM   LADDR   LPORT   RADDR   RPORT   TX_KB   RX_KB   MS
```

| Column | Meaning |
|---|---|
| `PID` | Process ID that owns the connection |
| `COMM` | Process or command name |
| `LADDR` | Local IP address |
| `LPORT` | Local TCP port |
| `RADDR` | Remote IP address |
| `RPORT` | Remote TCP port |
| `TX_KB` | Data transmitted, in KB |
| `RX_KB` | Data received, in KB |
| `MS` | Connection lifetime, in milliseconds |

**Example interpretation (illustrative):**

```text
PID    COMM   LADDR        LPORT  RADDR          RPORT  TX_KB  RX_KB  MS
2450   dnf    10.0.0.10    43512  52.14.50.151   443    18     820    3200
```

This example suggests that the `dnf` process used a TCP connection from local port `43512` to remote port `443` (HTTPS). It received about 820 KB and the connection lasted about 3,200 milliseconds.

The IP address and values above are examples only; your system will show its own connections.

### 3.3 Monitor file activity: `filetop`

In the **filetop** terminal, run:

```bash
/usr/share/bcc/tools/filetop
```

`filetop` is similar to the Linux `top` command, but it focuses on file activity. It refreshes periodically and shows which processes are reading or writing files.

Common fields include:

| Field | Meaning |
|---|---|
| `COMM` | Process name |
| `T` | File type |
| `READS` | Number of read operations |
| `WRITES` | Number of write operations |
| `R_KB` | Amount of data read, in KB |
| `W_KB` | Amount of data written, in KB |
| `FILE` | File being accessed |

**Why is this useful?**

During a package update, `dnf` and related processes read package data and write files to disk. `filetop` helps you identify which processes and files are generating that activity.

**Example:** If `dnf` appears with a high `READS` count and a large `R_KB` value, it is reading files. If a process has a high `WRITES` count or `W_KB` value, it is writing data.

The exact columns can vary by BCC version.

### 3.4 Monitor slow XFS operations: `xfsslower`

In the **xfsslower** terminal, run:

```bash
/usr/share/bcc/tools/xfsslower
```

By default, this tool reports XFS operations that take longer than its default latency threshold (commonly 10 ms).

Typical output includes:

- `COMM`: process name
- `PID`: process ID
- `LAT(ms)`: operation latency in milliseconds
- `FILENAME`: file involved in the operation

**Why is this useful?**

It helps identify slow file operations on XFS filesystems. You may see little or no output while the system is idle. During a workload such as package installation, disk activity may increase and slow operations may become visible.

**Example interpretation (illustrative):**

```text
COMM   PID   LAT(ms)   FILENAME
dnf    2450  35.20     packages.db-journal
```

This indicates that an XFS operation involving `packages.db-journal` took approximately 35.20 milliseconds. It is a clue for further storage investigation, not proof by itself that the disk is faulty.

> If your filesystem is not XFS, `xfsslower` will not provide the expected XFS observations.

### 3.5 Monitor memory page-cache activity: `cachestat`

In the **cachestat** terminal, run:

```bash
/usr/share/bcc/tools/cachestat
```

This tool reports activity involving the Linux page cache, which keeps recently accessed file data in memory to reduce the need for storage reads.

The output commonly includes fields such as:

- `HITS`: cache accesses served from memory
- `MISSES`: cache accesses that were not served from the page cache
- `Dirtied`: pages made dirty by writes
- `Buffers_MB`: buffer memory
- `Cached_MB`: cached memory

The precise columns depend on the BCC version.

**Simple example:**

- A cache **hit** means the requested data was available in memory.
- A cache **miss** means the access was not satisfied by the page cache and may require I/O from storage.

A high number of misses during a workload can indicate more file data is being fetched rather than served from cache. Interpret the counters over time and alongside actual disk activity; a miss count alone does not prove a storage bottleneck.

---

## 4. Run the sample workload: `dnf update`

Once all monitoring tools are running in their own terminals, switch to the **dnf** terminal.

Run:

```bash
dnf update -y
```

This updates installed packages without prompting for confirmation. It may take some time and will use network, CPU, memory, and storage resources.

### Phase A: Repository metadata and package downloads

At the beginning, `dnf` checks repository metadata and downloads required information and packages.

Watch these terminals:

**`gethostlatency`**

- Look for repository hostnames, for example `cdn.redhat.com`.
- Note the reported lookup latency.
- A slow lookup may contribute to connection setup delay, but it is only one part of the overall network path.

**`tcplife`**

- Look for connections owned by `dnf` or related processes.
- HTTPS connections commonly use remote port `443`.
- Review the remote IP, received data (`RX_KB`), and connection lifetime (`MS`).

**Example:** If `tcplife` shows a `dnf` connection to remote port `443` with a large `RX_KB` value, that connection likely received data over HTTPS. Use the process name and timing as clues; do not assume every port-443 connection belongs to the same repository.

### Phase B: Package installation and file operations

After downloading, `dnf` installs or updates packages and may remove older package versions. This creates file and storage activity.

Watch these terminals:

**`filetop`**

- Look for `dnf`, `rpm`, or other package-management processes.
- Observe read and write counts.
- Check which files are being accessed and how much data is read or written.

**`xfsslower`**

- Look for operations exceeding the latency threshold.
- Pay attention to `LAT(ms)` and `FILENAME`.
- For example, an operation taking 567 ms is substantially slower than one taking 10 ms and may be worth investigating if it occurs repeatedly.

**`cachestat`**

- Watch how cache hits and misses change during installation.
- Misses may increase when the workload accesses file data that is not already cached.
- When the workload becomes idle, activity may reduce. Do not expect every counter to become zero immediately; the output is a stream of measurements and cached data can remain in memory.

---

## 5. How to use the results to troubleshoot

The tools are most useful when you correlate their output rather than looking at one command in isolation.

| What you notice | Possible area to investigate | Useful tool |
|---|---|---|
| Hostname lookups take a long time | DNS configuration, resolver reachability, or DNS response time | `gethostlatency` |
| TCP connections take a long time or transfer little data | Network path, remote service, or connection behavior | `tcplife` |
| One process reads or writes many files | Application or package-manager file workload | `filetop` |
| Repeated high-latency XFS operations | Filesystem or underlying storage performance | `xfsslower` |
| Cache misses increase significantly during file access | Working-set size and storage reads; correlate with disk metrics | `cachestat` |

### Example: Investigating a slow package update

Imagine `dnf update` is taking longer than expected:

1. `gethostlatency` shows hostname lookups completing quickly. DNS is less likely to be the main delay.
2. `tcplife` shows a long-lived HTTPS connection. Investigate the network path and repository response.
3. `filetop` shows heavy file reads and writes during installation. This confirms substantial file activity.
4. `xfsslower` repeatedly reports high-latency operations on the same filesystem. Investigate storage latency and the virtual machine's backing storage.
5. `cachestat` shows cache misses increasing while the installation is reading data. Correlate this with disk I/O metrics.

This workflow narrows down where to investigate. It does not, by itself, prove the root cause. Also check system-level metrics such as `iostat`, CPU usage, memory pressure, and network statistics when needed.

---

## 6. Quick command reference

Run each monitoring command in its own terminal and leave it active:

```bash
# Inspect loaded eBPF programs
bpftool prog list
bpftool prog list | grep -A 3 cgroup

# Hostname lookup latency
/usr/share/bcc/tools/gethostlatency

# TCP connection activity
/usr/share/bcc/tools/tcplife

# File read/write activity
/usr/share/bcc/tools/filetop

# Slow XFS operations
/usr/share/bcc/tools/xfsslower

# Page-cache statistics
/usr/share/bcc/tools/cachestat

# Sample workload (run in the dnf terminal)
dnf update -y
```

## 7. Important notes

- These commands assume the BCC tools are installed at `/usr/share/bcc/tools/` and that the lab system has the required kernel support.
- Run the tools with sufficient privileges, usually as `root` or with `sudo`.
- eBPF tools can require compatible kernel features and permissions. If a tool fails to attach, check its error message, kernel support, and security restrictions.
- `xfsslower` is specifically for XFS; it is not a general-purpose slow-I/O monitor for every filesystem.
- The sample IP addresses, process IDs, filenames, and measurements in this guide are illustrative. Your output will differ.
- When finished, stop each foreground monitoring tool with **Ctrl+C** in its terminal. This stops the monitoring process; it does not uninstall BCC or remove eBPF support from the system.

## 8. Key takeaway

eBPF and BCC provide visibility into kernel-level activity while applications are running. In this lab:

- `gethostlatency` helps you observe hostname lookups.
- `tcplife` helps you observe TCP connections.
- `filetop` helps you observe file reads and writes.
- `xfsslower` helps you find slow XFS operations.
- `cachestat` helps you understand page-cache activity.

By observing all five tools while `dnf update` runs, you can build a clearer picture of how an application uses the network, filesystem, storage, and memory.
