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
{
  mv etcd etcdctl /usr/local/bin/
}
```

**Why:** This moves the etcd and etcdctl binaries from the home directory (where they were copied) to `/usr/local/bin/`, making them executable from anywhere on the system. Installing binaries to a standard location like `/usr/local/bin/` ensures they are on the system PATH, which is necessary because the systemd unit file will reference `etcd` without a full path .

### Configure the etcd Server

```bash
{
  mkdir -p /etc/etcd /var/lib/etcd
  chmod 700 /var/lib/etcd
  cp ca.crt kube-api-server.key kube-api-server.crt \
    /etc/etcd/
}
```

**Why:** This creates the configuration and data directories for etcd, sets restrictive permissions on the data directory (only the root user can access it), and copies the CA certificate, API server certificate, and API server private key to `/etc/etcd/`. The certificate and key are needed because etcd uses mutual TLS authentication to secure communication with the API server, and using the API server's certificate for etcd ensures that only the API server can access the database . The `chmod 700` on `/var/lib/etcd` protects sensitive cluster state data from unauthorized access.

Each etcd member must have a unique name within an etcd cluster. Set the etcd name to match the hostname of the current compute instance:

Create the `etcd.service` systemd unit file:

```bash
mv etcd.service /etc/systemd/system/
```

**Why:** This moves the systemd unit file from the home directory to `/etc/systemd/system/`, the standard location for system-wide service definitions. The service file contains the `ExecStart` command with all etcd configuration flags, including the name, data directory, TLS certificate paths, and client/peer listening addresses . Moving it to the systemd directory registers it with systemd, allowing the service to be managed with `systemctl` commands .

### Start the etcd Server

```bash
{
  systemctl daemon-reload
  systemctl enable etcd
  systemctl start etcd
}
```

**Why:** This sequence loads the new etcd systemd service definition so systemd recognizes it, enables the service to start automatically on boot, and immediately starts the etcd server. The `daemon-reload` is necessary because the service unit file was just moved into the systemd directory, and `enable` creates the necessary symlinks so etcd runs when the server reboots. Starting etcd now initializes the database and makes it available for the Kubernetes API server to connect to .

## Verification

List the etcd cluster members:

```bash
etcdctl member list
```

**Why:** This uses the etcdctl client to display the members of the etcd cluster, verifying that the etcd server is running and responding to client requests. In this single-node setup, it returns information about the server including its ID, status, peer URL, and client URL . This verification step is critical before proceeding to bootstrap the Kubernetes control plane because the API server depends entirely on a working etcd database.

```text
6702b0a34e2cfd39, started, controller, http://127.0.0.1:2380, http://127.0.0.1:2379, false
```

Next: [Bootstrapping the Kubernetes Control Plane](08-bootstrapping-kubernetes-controllers.md)