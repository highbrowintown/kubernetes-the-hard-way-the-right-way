# Bootstrapping the Kubernetes Worker Nodes

In this lab you will bootstrap two Kubernetes worker nodes. The following components will be installed:

| Component | Purpose |
|-----------|---------|
| [runc](https://github.com/opencontainers/runc) | The OCI-compliant container runtime that actually executes containers. It creates and manages the container namespaces, cgroups, and filesystem isolation. |
| [container networking plugins](https://github.com/containernetworking/cni) | CNI plugins provide the network connectivity for pods. They create virtual network interfaces, assign IP addresses, and configure routing so pods can communicate with each other and the outside world. |
| [containerd](https://github.com/containerd/containerd) | The high-level container runtime that manages the complete container lifecycle: image pulling, storage management, and container execution (delegating to runc). The kubelet communicates with containerd to create and manage pods. |
| [kubelet](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet) | The primary agent that runs on each worker node. It registers the node with the API server, receives pod specifications, and ensures the containers described in those pods are running and healthy. |
| [kube-proxy](https://kubernetes.io/docs/concepts/cluster-administration/proxies) | Maintains network rules on each node that enable Kubernetes Services to work. It handles packet forwarding for ClusterIP, NodePort, and LoadBalancer service types, allowing pods to be reached via stable virtual IPs. |

## Prerequisites

The commands in this section must be run from the `jumpbox`.

Copy the Kubernetes binaries and systemd unit files to each worker instance:

```bash
for HOST in node-0 node-1; do
  SUBNET=$(grep ${HOST} machines.txt | cut -d " " -f 4)
  sed "s|SUBNET|$SUBNET|g" \
    configs/10-bridge.conf > 10-bridge.conf

  sed "s|SUBNET|$SUBNET|g" \
    configs/kubelet-config.yaml > kubelet-config.yaml

  scp 10-bridge.conf kubelet-config.yaml \
  root@${HOST}:~/
done
```

**Why this is needed:** Each worker node needs a unique pod subnet so pods across nodes don't have IP address conflicts. This loop reads the subnet already assigned to each node from `machines.txt`, substitutes it into the `10-bridge.conf` and `kubelet-config.yaml` templates in place of the `SUBNET` placeholder, and copies the resulting node-specific files over before any of the other configuration.

```bash
for HOST in node-0 node-1; do
  scp \
    downloads/worker/* \
    downloads/client/kubectl \
    configs/99-loopback.conf \
    configs/containerd-config.toml \
    configs/kube-proxy-config.yaml \
    units/containerd.service \
    units/kubelet.service \
    units/kube-proxy.service \
    root@${HOST}:~/
done
```

**Why this is needed:** This copies the worker binaries (containerd, runc, kubelet, kube-proxy, crictl), `kubectl` for local debugging on the node itself, configuration files for each component, and the systemd unit files that define how each service starts, restarts, and depends on the others.

```bash
for HOST in node-0 node-1; do
  scp \
    downloads/cni-plugins/* \
    root@${HOST}:~/cni-plugins/
done
```

**Why this is needed:** CNI plugins are copied separately because they are a distinct set of third-party binaries that implement different network behaviors (bridge, loopback, host-local IPAM, etc.), and they land in their own `~/cni-plugins/` directory on the target so the later install step can move the whole set into `/opt/cni/bin/` in one action. The kubelet invokes these plugins whenever it sets up or tears down a pod's network.

The commands in the next section must be run on each worker instance: `node-0`, `node-1`. Login to the worker instance using the `ssh` command. Example:

```bash
ssh root@node-0
```

## Provisioning a Kubernetes Worker Node

Install the OS dependencies:

```bash
apt-get update
```

**Why:** Refreshes the local package index before installing, so the packages below are pulled at their current available versions rather than a stale cached index.

```bash
apt-get -y install socat conntrack ipset kmod
```

| Package | Why It's Needed |
|---------|-----------------|
| `socat` | Enables `kubectl port-forward` to work. Port forwarding creates a tunnel between your local machine and a pod, and socat handles the bidirectional data streaming for this tunnel. |
| `conntrack` | Required by kube-proxy for connection tracking. Kubernetes Services rely on conntrack to track network connections and properly route return traffic back to the correct pod. |
| `ipset` | Used by kube-proxy to efficiently manage large sets of IP addresses for Service endpoints. Instead of thousands of individual iptables rules, ipset allows matching against a set of IPs in a single rule, dramatically improving performance. |
| `kmod` | Provides utilities for loading and managing Linux kernel modules. Required to load modules like `br-netfilter` that Kubernetes networking depends on. |

### Disable Swap

Kubernetes has limited support for the use of swap memory, as it is difficult to provide guarantees and account for pod memory utilization when swap is involved.

**Why this is critical:** Kubernetes uses `cgroups` to enforce memory limits on pods. When swap is enabled, a pod's memory could be swapped to disk, making it impossible to accurately measure or limit memory usage. A pod could appear to use 100MB but actually have 500MB swapped out, defeating resource guarantees and potentially causing unexpected OOM kills when swapped pages are accessed. This is why the kubelet will fail to start if swap is detected.

Verify if swap is disabled:

```bash
swapon --show
```

If output is empty then swap is disabled. If swap is enabled run the following command to disable swap immediately:

```bash
swapoff -a
```

To ensure swap remains off after reboot consult your Linux distro documentation.

### Create the Installation Directories

```bash
mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes
```

| Directory | Purpose |
|-----------|---------|
| `/etc/cni/net.d` | Stores CNI network configuration files. The kubelet reads these to determine how to configure pod networking. |
| `/opt/cni/bin` | Contains the CNI plugin binaries. The kubelet executes these binaries when setting up or tearing down pod networks. |
| `/var/lib/kubelet` | The kubelet's data directory. Stores configuration files, pod manifests, and plugin data. |
| `/var/lib/kube-proxy` | Stores kube-proxy configuration and state files. |
| `/var/lib/kubernetes` | General Kubernetes data directory for certificates and other shared configuration. |
| `/var/run/kubernetes` | Runtime directory for Kubernetes components, used for sockets and PID files. |

### Install the Worker Binaries

```bash
mv crictl kube-proxy kubelet runc \
  /usr/local/bin/
```

**Why this location:** `/usr/local/bin/` is the standard location for locally installed user binaries and is in the default PATH, so `crictl`, `kube-proxy`, `kubelet`, and `runc` become runnable from anywhere without conflicting with any distro-packaged version.

```bash
mv containerd containerd-shim-runc-v2 containerd-stress /bin/
```

**Why this location:** `/bin/` is used for the containerd components because they are fundamental runtime binaries that other services depend on early in boot.

```bash
mv cni-plugins/* /opt/cni/bin/
```

**Why this location:** `/opt/cni/bin/` is the default location the kubelet's CNI integration looks for plugin binaries, so they must land here to be discoverable when a pod network is created or torn down.

| Binary | Purpose |
|--------|---------|
| `crictl` | CLI tool for interacting with CRI-compatible container runtimes. Useful for debugging container issues on the node. |
| `kube-proxy` | The network proxy daemon that implements Service abstraction. |
| `kubelet` | The node agent that communicates with the API server and manages containers. |
| `runc` | The OCI runtime that creates and runs containers. |
| `containerd` | The container runtime daemon that manages images and containers. |
| `containerd-shim-runc-v2` | A shim process that sits between containerd and runc, allowing containerd to be restarted without killing running containers. |
| `containerd-stress` | Testing tool for containerd (not strictly required for production). |

### Configure CNI Networking

Create the `bridge` network configuration file:

```bash
mv 10-bridge.conf 99-loopback.conf /etc/cni/net.d/
```

**Why two files:** CNI processes configuration files in alphabetical order. `10-bridge.conf` creates a bridge network interface and assigns an IP to the pod. `99-loopback.conf` adds a loopback interface (`lo`) inside the pod so localhost networking works. The numbering ensures the bridge is configured first, then loopback.

**Why a bridge network:** Each pod gets its own network namespace. The bridge plugin creates a virtual ethernet pair, one end in the pod and one end connected to a bridge on the host. This gives each pod a unique IP on the node's pod subnet and allows communication with other pods on the same node and across nodes (with additional routing).

To ensure network traffic crossing the CNI `bridge` network is processed by `iptables`, load and configure the `br-netfilter` kernel module:

```bash
modprobe br-netfilter
```

**Why:** Loads the `br-netfilter` kernel module immediately for the current session. By default, Linux bridges bypass iptables for performance. Kubernetes Services rely on iptables rules to implement ClusterIP routing, so without `br-netfilter`, traffic from a pod to a Service IP would bypass iptables and never reach kube-proxy's rules, breaking Service networking entirely.

```bash
echo "br-netfilter" >> /etc/modules-load.d/modules.conf
```

**Why:** Registers `br-netfilter` to load automatically on every future boot, so this fix survives a reboot rather than only applying to the current session.

```bash
echo "net.bridge.bridge-nf-call-iptables = 1" \
  >> /etc/sysctl.d/kubernetes.conf
```

**Why:** Persists the sysctl setting that tells the kernel to route IPv4 bridged traffic through iptables, which kube-proxy's Service rules depend on.

```bash
echo "net.bridge.bridge-nf-call-ip6tables = 1" \
  >> /etc/sysctl.d/kubernetes.conf
```

**Why:** Same setting as above, but for IPv6 bridged traffic, kept as a separate sysctl key since IPv4 and IPv6 bridge-netfilter behavior are controlled independently.

```bash
sysctl -p /etc/sysctl.d/kubernetes.conf
```

**Why:** Applies the settings just written to `/etc/sysctl.d/kubernetes.conf` immediately, without requiring a reboot.

### Configure containerd

Install the `containerd` configuration files:

```bash
mkdir -p /etc/containerd/
```

**Why:** Creates the directory containerd expects its configuration file to live in.

```bash
mv containerd-config.toml /etc/containerd/config.toml
```

**Why containerd needs this configuration:** `config.toml` specifies settings such as: the CRI plugin being enabled (required for kubelet communication), the snapshotter to use (e.g., overlayfs for layered image storage), the path to the `runc` runtime binary, the cgroup driver (which must match the kubelet's), and logging/metrics behavior.

```bash
mv containerd.service /etc/systemd/system/
```

**Why the systemd service file matters:** Defines how containerd is managed by systemd, including startup order, restart policy, and command-line flags. This ensures containerd starts automatically on boot and restarts if it crashes.

### Configure the Kubelet

Create the `kubelet-config.yaml` configuration file:

```bash
mv kubelet-config.yaml /var/lib/kubelet/
```

**Why kubelet-config.yaml is needed:** This `KubeletConfiguration` file defines the pod CIDR for this node (must match the subnet assigned earlier), the cgroup driver (must match containerd, typically `systemd`), the cluster DNS IP and domain used inside pods, authentication and authorization settings for the kubelet's own API, and related behavioral tuning.

```bash
mv kubelet.service /etc/systemd/system/
```

**Why:** Installs the systemd unit that tells systemd how to start, restart, and supervise the kubelet process, and which flags (including the path to `kubelet-config.yaml`) to launch it with.

**Why the kubelet needs its configuration and certificates:** The kubelet must authenticate to the API server to register itself as a node, report node status (capacity, conditions, addresses), retrieve the pod specifications assigned to it, and report pod status back to the control plane.

### Configure the Kubernetes Proxy

```bash
mv kube-proxy-config.yaml /var/lib/kube-proxy/
```

**Why kube-proxy-config.yaml is needed:** This `KubeProxyConfiguration` file defines the kubeconfig used to authenticate to the API server, the mode of operation (iptables, ipvs, or userspace, typically `iptables`), the cluster CIDR range used to distinguish internal from external traffic, and related bind-address/health-check settings.

```bash
mv kube-proxy.service /etc/systemd/system/
```

**Why:** Installs the systemd unit that governs how kube-proxy starts and restarts on this node.

**Why kube-proxy needs API server access:** kube-proxy watches the API server for Service objects (to create ClusterIP rules), Endpoints objects (to know which pods back each Service), and Node objects (to configure NodePort rules).

### Start the Worker Services

```bash
systemctl daemon-reload
```

**Why:** Tells systemd to re-read unit files on disk, picking up the `containerd.service`, `kubelet.service`, and `kube-proxy.service` files just copied into place. Without this, systemd would still be working from its previously cached view of available units.

```bash
systemctl enable containerd kubelet kube-proxy
```

**Why:** Creates the symlinks that make these three services start automatically on every future boot, not just this session.

```bash
systemctl start containerd kubelet kube-proxy
```

**Why starting them together works:** systemd resolves the dependency ordering declared in the unit files themselves (for example, `kubelet.service` waiting on `containerd.service`), so listing all three in one command is safe. Starting containerd first in practice ensures its CRI socket is available by the time kubelet attempts to create containers, and kubelet in turn needs to be running before kube-proxy's Service rules are meaningful.

**What happens during startup:** containerd starts and begins listening on its CRI socket; kubelet starts, connects to the API server, registers the node, and begins polling for pods assigned to it; kube-proxy starts, connects to the API server, and begins watching Services and Endpoints to configure iptables rules.

Check if the kubelet service is running:

```bash
systemctl is-active kubelet
```

```text
active
```

**If not active:** Check logs with `journalctl -u kubelet -f` to diagnose issues. Common problems include an incorrect kubeconfig, a mismatched cgroup driver between kubelet and containerd, or missing/expired certificates.

Be sure to complete the steps in this section on each worker node, `node-0` and `node-1`, before moving on to the next section.

## Verification

Run the following commands from the `jumpbox` machine.

List the registered Kubernetes nodes:

```bash
ssh root@server \
  "kubectl get nodes \
  --kubeconfig admin.kubeconfig"
```

```text
NAME     STATUS   ROLES    AGE    VERSION
node-0   Ready    <none>   1m     v1.32.3
node-1   Ready    <none>   10s    v1.32.3
```

**What "Ready" means:** A node reaches `Ready` status once the kubelet has successfully registered it with the API server, the kubelet's node-status heartbeat is being received by the control plane, all required node conditions (`MemoryPressure`, `DiskPressure`, `PIDPressure`, `NetworkUnavailable`) report `False`, and `NetworkUnavailable` in particular only clears once the CNI plugin has reported the node's pod network is ready.

**Why ROLES shows `<none>`:** No node labels (such as `node-role.kubernetes.io/worker=`) have been applied. Labels are optional; the cluster functions correctly without them, but they are useful for scheduling constraints and for grouping nodes in `kubectl` output.

---

Next: [Configuring kubectl for Remote Access](10-configuring-kubectl.md)