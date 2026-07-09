# Set Up The Jumpbox

In this lab you will set up one of the four machines to be a `jumpbox`. This machine will be used to run commands throughout this tutorial. While a dedicated machine is being used to ensure consistency, these commands can also be run from just about any machine including your personal workstation running macOS or Linux.

Think of the `jumpbox` as the administration machine that you will use as a home base when setting up your Kubernetes cluster from the ground up. Before we get started we need to install a few command line utilities and clone the Kubernetes The Hard Way git repository, which contains some additional configuration files that will be used to configure various Kubernetes components throughout this tutorial.

**Why a dedicated jumpbox:** Kubernetes the Hard Way builds every cluster component manually, without kubeadm or any installer. That means certificates, kubeconfigs, and systemd unit files all get generated on one machine and then copied out to `server`, `node-0`, and `node-1`. A single jumpbox keeps that generation and distribution process consistent, and prevents drift that would happen if you generated certs from different machines with different tool versions.

Log in to the `jumpbox`:

```bash
ssh root@jumpbox
```

**Why:** All subsequent commands in this tutorial assume they run from this shell session, in this working directory. Logging in here first anchors that context.

All commands will be run as the `root` user. This is being done for the sake of convenience, and will help reduce the number of commands required to set everything up.

**Why root:** Later steps write to `/etc/`, move binaries into `/usr/local/bin/`, and copy certificate/key material with restrictive ownership. Running as root avoids interrupting the tutorial with `sudo` on every line, though in production you would scope this down.

### Install Command Line Utilities

Now that you are logged into the `jumpbox` machine as the `root` user, you will install the command line utilities that will be used to preform various tasks throughout the tutorial.

```bash
apt-get update
```

**Why:** `apt-get update` is run first because the package index on a fresh machine may be stale, and installing without updating can pull outdated or unavailable package versions. This ensures the package lists are current before trying to install dependencies.

```bash
apt-get -y install wget curl vim openssl git
```

**Why each tool is needed:**
- `wget` — downloads the Kubernetes and etcd binary releases in the next section.
- `curl` — used later for talking to the Kubernetes API server directly and for smoke-testing endpoints.
- `vim` — for editing configuration files, systemd unit files, and YAML manifests on the jumpbox and remote machines.
- `openssl` — used to generate and inspect the TLS certificates and private keys that secure communication between every Kubernetes component (this is the "hard way" replacement for what `kubeadm` normally automates).
- `git` — required to clone the tutorial repository itself, which is the next step.

### Sync GitHub Repository

Now it's time to download a copy of this tutorial which contains the configuration files and templates that will be used build your Kubernetes cluster from the ground up. Clone the Kubernetes The Hard Way git repository using the `git` command:

```bash
git clone --depth 1 https://github.com/highbrowintown/kubernetes-the-hard-way-the-right-way.git
```

**Why:** The repository ships more than instructions — it contains the `downloads-amd64.txt`/`downloads-arm64.txt` binary manifests, Kubernetes config templates, and systemd unit templates referenced in later labs. Without cloning it, you'd have to hand-author all of those files yourself. `--depth 1` is used because you only need the latest snapshot of the repo, not its full commit history, which keeps the clone fast and small.

Change into the `kubernetes-the-hard-way` directory:

```bash
cd kubernetes-the-hard-way-the-right-way
```

This will be the working directory for the rest of the tutorial. If you ever get lost run the `pwd` command to verify you are in the right directory when running commands on the `jumpbox`:

```bash
pwd
```

**Why this matters:** Nearly every command from this point forward is relative to this directory (for example, referencing `downloads-$(dpkg --print-architecture).txt` or the `downloads/` folder). Running commands from the wrong directory is one of the most common sources of "file not found" errors in this tutorial, hence the explicit `pwd` checkpoint.

```text
/root/kubernetes-the-hard-way
```

### Download Binaries

In this section you will download the binaries for the various Kubernetes components. The binaries will be stored in the `downloads` directory on the `jumpbox`, which will reduce the amount of internet bandwidth required to complete this tutorial as we avoid downloading the binaries multiple times for each machine in our Kubernetes cluster.

**Why download once on the jumpbox instead of on each node:** Later labs distribute these binaries from the jumpbox to `server`, `node-0`, and `node-1` over the local network using `scp`. That's faster and more reliable than having each of the three cluster machines independently pull ~500MB from the internet, and it guarantees every machine ends up running the exact same binary version.

The binaries that will be downloaded are listed in either the `downloads-amd64.txt` or `downloads-arm64.txt` file depending on your hardware architecture, which you can review using the `cat` command:

```bash
cat downloads-$(dpkg --print-architecture).txt
```

**Why the architecture check:** `dpkg --print-architecture` detects whether the machine is `amd64` or `arm64` so the correct binary manifest (and correct binary builds later) are used. Kubernetes and etcd release separate builds per CPU architecture, and using the wrong one will fail to execute at all.

Download the binaries into a directory called `downloads` using the `wget` command:

```bash
wget -q --show-progress \
  --https-only \
  --timestamping \
  -P downloads \
  -i downloads-$(dpkg --print-architecture).txt
```

**Why these flags:**
- `--https-only` — ensures binaries are only ever fetched over encrypted connections, avoiding tampering in transit.
- `--timestamping` — skips re-downloading a file if the local copy is already up to date, which matters if you rerun this step after a failure partway through.
- `-P downloads` — puts everything in one directory so later extraction/move commands can rely on a known, fixed location.
- `-i downloads-...txt` — reads the list of URLs to fetch from the manifest file instead of typing each one out.

Depending on your internet connection speed it may take a while to download over `500` megabytes of binaries, and once the download is complete, you can list them using the `ls` command:

```bash
ls -oh downloads
```

Extract the component binaries from the release archives and organize them under the `downloads` directory. This is done as a series of individual steps below.

```bash
ARCH=$(dpkg --print-architecture)
```

**Why:** Detects the CPU architecture (`amd64` or `arm64`) of the jumpbox so the correct binary build is extracted and moved in every line that follows. Kubernetes, etcd, containerd, and crictl all publish separate release archives per architecture; using the wrong one produces a binary that simply will not execute.



```bash
mkdir -p downloads/{client,cni-plugins,controller,worker}
```

**Why:** Creates four destination folders that map to the four roles binaries will be distributed into: tools run from the jumpbox (`client`), pod networking plugins (`cni-plugins`), control-plane processes (`controller`), and node-level components (`worker`).



```bash
tar -xvf downloads/crictl-v1.32.0-linux-${ARCH}.tar.gz \
  -C downloads/worker/
```

**Why `crictl`:** `crictl` is a command-line interface for CRI-compatible container runtimes, built so operators can inspect and debug containers, pods, and images directly at the container-runtime level, independent of `kubectl` or the API server. It gives direct access to the container runtime interface (CRI), which is invaluable for debugging issues that occur below the Kubernetes abstraction layer, such as container startup failures, image pull problems, and runtime configuration issues. This is what lets you troubleshoot a node when the API server itself is unreachable, since it speaks directly to the runtime rather than through the cluster control plane. It goes into `worker/` because it only makes sense on a machine actually running containerd — `node-0` and `node-1`.

```bash
tar -xvf downloads/containerd-2.1.0-beta.0-linux-${ARCH}.tar.gz \
  --strip-components 1 \
  -C downloads/worker/
```

**Why `containerd` and `--strip-components 1`:** `containerd` is the actual container runtime — the daemon that pulls images, manages container lifecycle, and exposes the CRI socket that both the kubelet and `crictl` talk to. `--strip-components 1` drops the top-level folder the archive extracts to (something like `containerd-2.1.0-beta.0/`) so the binaries land directly in `downloads/worker/` instead of a nested subdirectory. This is a per-node runtime, not something the control plane or jumpbox needs, hence `worker/`.

```bash
tar -xvf downloads/cni-plugins-linux-${ARCH}-v1.6.2.tgz \
  -C downloads/cni-plugins/
```

**Why CNI plugins get their own directory:** CNI plugins are the executables that actually give a pod a network interface and an IP address. They are not daemons or long-running services — they are executables that the container runtime invokes at specific points in a container's lifecycle: when a pod starts, the runtime calls CNI plugins to configure networking, and when a pod terminates, the runtime calls CNI plugins again to clean up. Main plugins create network interfaces (bridge, ipvlan, macvlan, ptp) while IPAM plugins assign IP addresses to those interfaces. containerd (via the kubelet) invokes these plugins whenever a pod is created or torn down, which is why they get their own directory rather than being lumped into `worker/` — later in the tutorial they're distributed to a CNI-specific path (`/opt/cni/bin`) on the node rather than alongside the other worker binaries.

```bash
tar -xvf downloads/etcd-v3.6.0-rc.3-linux-${ARCH}.tar.gz \
  -C downloads/ \
  --strip-components 1 \
  etcd-v3.6.0-rc.3-linux-${ARCH}/etcdctl \
  etcd-v3.6.0-rc.3-linux-${ARCH}/etcd
```

**Why only these two files:** etcd is the distributed key-value store that holds all cluster state — every object the API server reads and writes ultimately lives here. This line pulls out exactly two files from the archive: `etcd` (the server binary itself) and `etcdctl` (its CLI client), ignoring everything else the release tarball contains (docs, other utilities). `--strip-components 1` again removes the versioned parent folder so both files land flat in `downloads/`, ready for the `mv` commands below to route them to their correct destinations.

```bash
mv downloads/{etcdctl,kubectl} downloads/client/
```

**Why these two go to `client/`:** `etcdctl` and `kubectl` are both client tools run from the jumpbox to talk to already-running services (etcd and the API server respectively) — neither needs to run persistently on a cluster machine.

```bash
mv downloads/{etcd,kube-apiserver,kube-controller-manager,kube-scheduler} \
  downloads/controller/
```

**Why these four go to `controller/`:** these are the control-plane processes that will run on the `server` machine:
- `etcd` — the cluster's state store, described above.
- `kube-apiserver` — the front door for the entire cluster; every `kubectl` command, every kubelet report, and every controller decision passes through it, and it is the only component that talks directly to etcd.
- `kube-controller-manager` — runs the reconciliation control loops (node lifecycle, replication, endpoints, etc.) that continuously drive actual cluster state toward the desired state stored in etcd.
- `kube-scheduler` — watches for newly created pods with no assigned node and decides which node they should run on.

```bash
mv downloads/{kubelet,kube-proxy} downloads/worker/
```

**Why these two go to `worker/`:**
- `kubelet` — the primary node agent; it registers the node with the cluster, starts pods by talking to containerd through the CRI, and reports node/pod health back to the API server.
- `kube-proxy` — runs on every node and implements the Service abstraction by maintaining the network rules that forward traffic to the right pod backing a Service.

Both are per-node daemons, so they belong alongside kubelet's other runtime dependencies (containerd, crictl, runc, CNI plugins).

```bash
mv downloads/runc.${ARCH} downloads/worker/runc
```

**Why `runc` and the rename:** `runc` is the low-level OCI runtime that actually creates and runs the container process (namespaces, cgroups, etc.); containerd shells out to `runc` to do this final step rather than implementing it itself. It is renamed from `runc.${ARCH}` (the architecture-suffixed release filename) to the plain `runc` that containerd expects to find on disk, and it lands in `worker/` since it is a dependency of the per-node container runtime.

**Why this organization matters overall:**
- `client/` (`kubectl`, `etcdctl`) — tools run *from* the jumpbox to talk to the cluster and to etcd; they don't need to live on the cluster nodes themselves.
- `controller/` (`etcd`, `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`) — the actual control plane processes, destined for the `server` machine.
- `worker/` (`kubelet`, `kube-proxy`, `containerd`, `crictl`, `runc`) — everything a node needs to actually run pods. These go to `node-0` and `node-1`.
- `cni-plugins/` — binaries that implement pod networking according to the CNI spec; the kubelet invokes these when starting a pod's network namespace.

Splitting into these four directories now means the distribution step later (copying files to each of the three machines) only has to `scp` the directory relevant to that machine's role, instead of hand-picking individual files.

```bash
rm -rf downloads/*gz
```

**Why:** Once binaries are extracted from the `.tar.gz`/`.tgz` archives, the compressed originals are just disk space with no further use — this cleans them up before you distribute the `downloads/` tree to other machines.

Make the binaries executable.

```bash
chmod +x downloads/{client,cni-plugins,controller,worker}/*
```

**Why:** Files extracted from an archive don't automatically retain (or may lack) the execute bit needed to run them as programs. Without this, later attempts to run `kubectl`, `etcd`, `kubelet`, etc. would fail with a permission error.

### Install kubectl

In this section you will install the `kubectl`, the official Kubernetes client command line tool, on the `jumpbox` machine. `kubectl` will be used to interact with the Kubernetes control plane once your cluster is provisioned later in this tutorial.

Use the `chmod` command to make the `kubectl` binary executable and move it to the `/usr/local/bin/` directory:

```bash
cp downloads/client/kubectl /usr/local/bin/
```

**Why `/usr/local/bin/` specifically:** That directory is on the default `$PATH` on virtually every Linux distribution and is conventionally reserved for binaries installed manually (outside the distro's package manager), so `kubectl` becomes runnable from anywhere without conflicting with any distro-packaged version.

At this point `kubectl` is installed and can be verified by running the `kubectl` command:

```bash
kubectl version --client
```

**Why verify here:** This confirms `kubectl` is on the `PATH`, executable, and correctly reporting its version — before you're several labs deep and trying to debug whether a later failure is a `kubectl` problem or a cluster problem. `--client` is used because there's no cluster running yet for it to talk to; this only checks the local binary.

```text
Client Version: v1.32.3
Kustomize Version: v5.5.0
```

At this point the `jumpbox` has been set up with all the command line tools and utilities necessary to complete the labs in this tutorial.

Next: [Provisioning Compute Resources](03-compute-resources.md)