# Bootstrapping the etcd Cluster

Kubernetes components are stateless and store cluster state in [etcd](https://github.com/etcd-io/etcd). In this lab you will bootstrap a single node etcd cluster.

## Prerequisites

Copy `etcd` binaries and systemd unit files to the `server` machine:

```bash
scp \
  downloads/controller/etcd \
  downloads/client/etcdctl \
  units/etcd.service \
  root@server:~/
```

**Why:** This copies the etcd server binary, the etcdctl command-line client, and the systemd service unit file from the jumpbox to the server machine. These files were downloaded during the prerequisites setup and must be present on the control plane node before etcd can be installed and configured. The etcd binary provides the database server that will store all Kubernetes cluster state, etcdctl is used for verifying the cluster is healthy, and the systemd unit defines how the service should run .

The commands in this lab must be run on the `server` machine. Login to the `server` machine using the `ssh` command. Example:

```bash
ssh root@server
```

**Why:** This establishes an SSH connection from the jumpbox to the server machine, where all subsequent commands in this section will be executed. etcd runs on the control plane node because the Kubernetes API server needs to communicate with it locally using the loopback address for secure, low-latency access to cluster state .

## Bootstrapping an etcd Cluster

### Install the etcd Binaries

Extract and install the `etcd` server and the `etcdctl` command line utility:

```bash
mv etcd etcdctl /usr/local/bin/
```

**Why:** This moves the etcd and etcdctl binaries from the home directory (where they were copied) to `/usr/local/bin/`, making them executable from anywhere on the system. Installing binaries to a standard location like `/usr/local/bin/` ensures they are on the system PATH, which is necessary because the systemd unit file will reference `etcd` without a full path .

### Configure the etcd Server

```bash
mkdir -p /etc/etcd /var/lib/etcd
```

**Why:** Creates the two directories etcd needs: `/etc/etcd/` for its TLS certificates and `/var/lib/etcd/` for its actual database files.

```bash
chmod 700 /var/lib/etcd
```

**Why:** Restricts the data directory so only the `root` user can read, write, or enter it, since this directory holds etcd's raw database files, including encrypted Secrets and the keys protecting them.

```bash
cp ca.crt kube-api-server.key kube-api-server.crt \
  /etc/etcd/
```

**Why:** Copies the CA certificate and the API server's certificate pair into `/etc/etcd/`. etcd uses `ca.crt` as its `--trusted-ca-file`/`--peer-trusted-ca-file` to validate incoming connections, and uses `kube-api-server.key`/`kube-api-server.crt` as its own `--cert-file`/`--key-file` (and `--peer-cert-file`/`--peer-key-file`) to serve TLS. Reusing the API server's certificate here, rather than issuing etcd a separate one, works because the API server is etcd's only real client in this single-node setup, and it saves a certificate-generation step. The API server in turn is configured with `--etcd-cafile`, `--etcd-certfile`, and `--etcd-keyfile` pointing at these same three files, so both sides authenticate each other using the shared CA — this is what makes the `https://127.0.0.1:2379` connection between them mutually authenticated rather than just encrypted.

This tutorial runs a single-node etcd cluster with the name hardcoded to `controller` directly in the `etcd.service` unit file below, so no separate step is needed to set it dynamically.

Create the `etcd.service` systemd unit file:

```bash
mv etcd.service /etc/systemd/system/
```

**Why:** This moves the systemd unit file from the home directory to `/etc/systemd/system/`, the standard location for system-wide service definitions. The service file contains the `ExecStart` command with all etcd configuration flags, including the name, data directory, TLS certificate paths, and client/peer listening addresses . Moving it to the systemd directory registers it with systemd, allowing the service to be managed with `systemctl` commands .

### Start the etcd Server

```bash
systemctl daemon-reload
```

**Why:** Tells systemd to re-read unit files on disk, picking up the `etcd.service` file just moved into `/etc/systemd/system/`.

```bash
systemctl enable etcd
```

**Why:** Creates the symlink that makes etcd start automatically on every future boot.

```bash
systemctl start etcd
```

**Why:** Starts etcd now, initializing its database and making it available for the Kubernetes API server to connect to on the next lab.

## Verification

List the etcd cluster members:

```bash
ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.crt \
  --cert=/etc/etcd/kube-api-server.crt \
  --key=/etc/etcd/kube-api-server.key
```

**Why:** Connects to etcd's client endpoint and lists its members, verifying the server is running and responding. Because `etcd.service` only listens for TLS client connections on `https://127.0.0.1:2379`, this command must present the same CA and certificate pair etcd was configured to trust — the plain `etcdctl member list` shown in earlier tutorial variants would fail here with a connection error, since there's no unencrypted listener to fall back to. This check is critical before proceeding to bootstrap the control plane, since the API server depends entirely on a working etcd database.

```text
6702b0a34e2cfd39, started, controller, https://127.0.0.1:2380, https://127.0.0.1:2379, false
```

Next: [Bootstrapping the Kubernetes Control Plane](08-bootstrapping-kubernetes-controllers.md)