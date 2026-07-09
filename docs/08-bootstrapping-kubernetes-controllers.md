# Bootstrapping the Kubernetes Control Plane

In this lab you will bootstrap the Kubernetes control plane. The following components will be installed on the `server` machine: Kubernetes API Server, Scheduler, and Controller Manager.

## Prerequisites

Connect to the `jumpbox` and copy Kubernetes binaries and systemd unit files to the `server` machine:

```bash
scp downloads/controller/kube-apiserver \
  downloads/controller/kube-controller-manager \
  downloads/controller/kube-scheduler \
  downloads/client/kubectl \
  units/kube-apiserver.service \
  units/kube-controller-manager.service \
  units/kube-scheduler.service \
  configs/kube-scheduler.yaml \
  configs/kube-apiserver-to-kubelet.yaml \
  root@server:~/
```

**Why:** This copies all Kubernetes control plane binaries, client tools, systemd service definitions, and configuration files from the jumpbox to the server machine . These components were downloaded during the prerequisites setup and must be present on the control plane node before installation; the service unit files define how each component should run as a systemd service, and the YAML files contain critical configuration such as scheduler policies and RBAC rules for kubelet authorization .

The commands in this lab must be run on the `server` machine. Login to the `server` machine using the `ssh` command. Example:

```bash
ssh root@server
```

**Why:** This establishes an SSH connection from the jumpbox to the server machine, where all subsequent control plane bootstrap commands will be executed. The control plane components must run on the designated server node because the API server needs to communicate locally with etcd and the other control plane services use the local kubeconfig files placed on this machine .

## Provision the Kubernetes Control Plane

Create the Kubernetes configuration directory:

```bash
mkdir -p /etc/kubernetes/config
```

**Why:** This creates the standard directory where Kubernetes configuration files, such as the scheduler configuration, will be stored. The control plane components expect their configuration files to be in well-known paths, and this directory convention helps organize multiple configuration artifacts on the control plane node .

### Install the Kubernetes Controller Binaries

Install the Kubernetes binaries:

```bash
mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
```

**Why:** This moves the Kubernetes binaries from the home directory (where they were copied) to `/usr/local/bin/`, making them executable from anywhere on the system. Installing binaries to a standard location on the system PATH ensures the systemd service files can reference each component by name without specifying full paths, and kubectl becomes available for cluster administration .

### Configure the Kubernetes API Server

```bash
mkdir -p /var/lib/kubernetes/
```

**Why:** Creates the directory where all control plane certificates, keys, and the encryption configuration will live. This is the fixed path the `kube-apiserver`, `kube-controller-manager`, and `kube-scheduler` systemd unit files all point to.

```bash
mv ca.crt ca.key kube-api-server.key kube-api-server.crt service-accounts.key service-accounts.crt encryption-config.yaml /var/lib/kubernetes/
```

**Why:** Moves every credential the control plane needs into that directory. `ca.crt` lets the API server validate client certificates presented by kubelets and other components; `kube-api-server.key`/`kube-api-server.crt` are the API server's own TLS pair for serving HTTPS; `service-accounts.key`/`service-accounts.crt` are used by `kube-controller-manager` to sign and verify service account tokens; and `encryption-config.yaml` tells the API server how to encrypt Secrets at rest in etcd. `ca.key` is included here not for the API server, but because `kube-controller-manager` (which also runs on this machine) uses it via `--cluster-signing-cert-file`/`--cluster-signing-key-file` to sign certificates issued through the CertificateSigningRequest API.

Create the `kube-apiserver.service` systemd unit file:

```bash
mv kube-apiserver.service /etc/systemd/system/kube-apiserver.service
```

**Why:** This moves the API server systemd unit file to the system services directory, registering it with systemd. The service file defines how the API server should run, including command-line arguments that specify certificate paths, authorization modes, admission plugins, etcd endpoints, and service cluster IP ranges . Moving this file to `/etc/systemd/system/` makes the API server manageable with standard systemctl commands.

### Configure the Kubernetes Controller Manager

Move the `kube-controller-manager` kubeconfig into place:

```bash
mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
```

**Why:** This places the controller manager's kubeconfig file in the Kubernetes data directory, where the controller manager service expects to find it. This kubeconfig contains the client certificate and cluster endpoint information needed for the controller manager to authenticate to the API server when performing its reconciliation loops, such as node lifecycle management, replication, and service account token generation .

Create the `kube-controller-manager.service` systemd unit file:

```bash
mv kube-controller-manager.service /etc/systemd/system/
```

**Why:** This moves the controller manager systemd unit file to the system services directory, registering it with systemd. The service file includes flags for leader election (in HA setups), certificate signing authority, and various controller-specific configuration options, enabling the controller manager to run as a managed background service .

### Configure the Kubernetes Scheduler

Move the `kube-scheduler` kubeconfig into place:

```bash
mv kube-scheduler.kubeconfig /var/lib/kubernetes/
```

**Why:** This places the scheduler's kubeconfig file in the Kubernetes data directory, where the scheduler service expects to find it. This kubeconfig contains the client certificate needed for the scheduler to authenticate to the API server when binding pods to nodes, which is essential for pod scheduling decisions .

Create the `kube-scheduler.yaml` configuration file:

```bash
mv kube-scheduler.yaml /etc/kubernetes/config/
```

**Why:** This moves the scheduler's configuration file to `/etc/kubernetes/config/`, the standard location for control plane component configurations. The YAML file defines the scheduler's policies, such as algorithm source, score strategies, and leader election behavior, which control how the scheduler selects nodes for pod placement .

Create the `kube-scheduler.service` systemd unit file:

```bash
mv kube-scheduler.service /etc/systemd/system/
```

**Why:** This moves the scheduler systemd unit file to the system services directory, registering it with systemd. The service file points to the scheduler's kubeconfig and configuration file, allowing the scheduler to run as a managed background service that watches the API server for unscheduled pods .

### Start the Controller Services

```bash
systemctl daemon-reload
```

**Why:** Tells systemd to re-read unit files on disk, picking up the three service files just copied into `/etc/systemd/system/`.

```bash
systemctl enable kube-apiserver kube-controller-manager kube-scheduler
```

**Why:** Creates the symlinks that make all three control plane services start automatically on every future boot.

```bash
systemctl start kube-apiserver kube-controller-manager kube-scheduler
```

**Why:** Starts all three now. Listing them together is safe because `kube-controller-manager` and `kube-scheduler` both depend on `kube-apiserver` being reachable to do their jobs, and systemd will bring them up in an order where the API server is available first.

> Allow up to 10 seconds for the Kubernetes API Server to fully initialize.

You can check if any of the control plane components are active using the `systemctl` command. For example, to check if the `kube-apiserver` fully initialized, and active, run the following command:

```bash
systemctl is-active kube-apiserver
```

**Why:** This checks whether the API server service is currently running and active, returning "active" if the process is up and responding. This quick status check helps verify that the service started successfully after the bootstrap process .

For a more detailed status check, which includes additional process information and log messages, use the `systemctl status` command:

```bash
systemctl status kube-apiserver
```

**Why:** This displays detailed service information including the process ID, memory usage, and recent log messages from the API server. This is useful for troubleshooting if the service failed to start or is behaving unexpectedly .

If you run into any errors, or want to view the logs for any of the control plane components, use the `journalctl` command. For example, to view the logs for the `kube-apiserver` run the following command:

```bash
journalctl -u kube-apiserver
```

**Why:** This displays the complete systemd journal logs for the API server service, which is essential for debugging startup failures, certificate issues, or etcd connection problems. The logs contain detailed error messages that help diagnose why the control plane components may not be functioning correctly .

### Verification

At this point the Kubernetes control plane components should be up and running. Verify this using the `kubectl` command line tool:

```bash
kubectl cluster-info --kubeconfig admin.kubeconfig
```

**Why:** This queries the API server using the admin kubeconfig to display cluster endpoint information, confirming the control plane is reachable and responding correctly .

```text
Kubernetes control plane is running at https://127.0.0.1:6443
```

## RBAC for Kubelet Authorization

In this section you will configure RBAC permissions to allow the Kubernetes API Server to access the Kubelet API on each worker node. Access to the Kubelet API is required for retrieving metrics, logs, and executing commands in pods.

> This tutorial sets the Kubelet `--authorization-mode` flag to `Webhook`. Webhook mode uses the [SubjectAccessReview](https://kubernetes.io/docs/reference/access-authn-authz/authorization/#checking-api-access) API to determine authorization.

The commands in this section will affect the entire cluster and only need to be run on the `server` machine.

```bash
ssh root@server
```

**Why:** This re-establishes the SSH connection to the server machine if it was lost or if you were working from the jumpbox. The RBAC configuration must be applied using the admin kubeconfig on the control plane node where the API server is running .

Create the `system:kube-apiserver-to-kubelet` [ClusterRole](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#role-and-clusterrole) with permissions to access the Kubelet API and perform most common tasks associated with managing pods:

```bash
kubectl apply -f kube-apiserver-to-kubelet.yaml --kubeconfig admin.kubeconfig
```

**Why:** This applies a ClusterRole and ClusterRoleBinding that grants the API server permission to access Kubelet APIs on worker nodes . The ClusterRole defines permissions for nodes/proxy, nodes/stats, nodes/log, nodes/spec, and nodes/metrics resources, which are needed for kubectl logs, kubectl exec, and metrics collection . This step is necessary because the API server authenticates to Kubelets as the `kubernetes` user using its client certificate, and without this RBAC configuration, operations like `kubectl logs` and `kubectl exec` would fail with permission errors .

### Verification

At this point the Kubernetes control plane is up and running. Run the following commands from the `jumpbox` machine to verify it's working:

Make a HTTP request for the Kubernetes version info:

```bash
curl --cacert ca.crt https://server.kubernetes.local:6443/version
```

**Why:** This makes an authenticated HTTPS request to the API server's version endpoint from the jumpbox, verifying that the API server is reachable and responding with the correct version information .

```json
{
  "major": "1",
  "minor": "32",
  "gitVersion": "v1.32.3",
  "gitCommit": "32cc146f75aad04beaaa245a7157eb35063a9f99",
  "gitTreeState": "clean",
  "buildDate": "2025-03-11T19:52:21Z",
  "goVersion": "go1.23.6",
  "compiler": "gc",
  "platform": "linux/arm64"
}
```

Next: [Bootstrapping the Kubernetes Worker Nodes](09-bootstrapping-kubernetes-workers.md)