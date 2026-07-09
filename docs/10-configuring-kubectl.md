# Configuring kubectl for Remote Access

In this lab you will generate a kubeconfig file for the `kubectl` command line utility based on the `admin` user credentials.

> Run the commands in this lab from the `jumpbox` machine.

## The Admin Kubernetes Configuration File

Each kubeconfig requires a Kubernetes API Server to connect to.

You should be able to ping `server.kubernetes.local` based on the `/etc/hosts` DNS entry from a previous lab.

```bash
curl --cacert ca.crt \
  https://server.kubernetes.local:6443/version
```

**Why:** This command verifies that the Kubernetes API server is reachable from the jumpbox machine and that the TLS certificate is trusted by using the CA certificate . This check confirms both the network connectivity and the hostname resolution before generating the kubeconfig, preventing configuration errors that would occur if the API server were unreachable.

```text
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

Generate a kubeconfig file suitable for authenticating as the `admin` user:

```bash
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.crt \
  --embed-certs=true \
  --server=https://server.kubernetes.local:6443
```

**Why:** Defines the cluster entry: the API server's remote address and the CA certificate to verify it. No `--kubeconfig` flag is given, which is what makes this write to the default location, `~/.kube/config`, rather than a named file like the server-side admin kubeconfig used. `server.kubernetes.local` is used here instead of `127.0.0.1` because this command runs from the jumpbox, not from the server itself, so it needs the API server's actual reachable hostname.

```bash
kubectl config set-credentials admin \
  --client-certificate=admin.crt \
  --client-key=admin.key
```

**Why:** Adds the admin user's client certificate and key as a named credential in the same default kubeconfig.

```bash
kubectl config set-context kubernetes-the-hard-way \
  --cluster=kubernetes-the-hard-way \
  --user=admin
```

**Why:** Creates a context tying the cluster and user together under one name, so `kubectl` knows which server to talk to using which identity.

```bash
kubectl config use-context kubernetes-the-hard-way
```

**Why:** Activates this context as the default, which is what lets every subsequent `kubectl` command in this tutorial run without needing `--kubeconfig` or `--context` flags.

The results of running the command above should create a kubeconfig file in the default location `~/.kube/config` used by the  `kubectl` commandline tool. This also means you can run the `kubectl` command without specifying a config.

## Verification

Check the version of the remote Kubernetes cluster:

```bash
kubectl version
```

**Why:** This command confirms that `kubectl` can communicate with the API server using the newly generated default kubeconfig, displaying both client and server version information. The server version should match the Kubernetes version installed during the control plane bootstrap, verifying successful connectivity .

```text
Client Version: v1.32.3
Kustomize Version: v5.5.0
Server Version: v1.32.3
```

List the nodes in the remote Kubernetes cluster:

```bash
kubectl get nodes
```

**Why:** This lists all worker nodes registered in the cluster, verifying that both `node-0` and `node-1` have successfully joined and are reporting as `Ready` . This confirms the worker node bootstrap process was successful and the cluster is fully operational with all expected nodes.

```text
NAME     STATUS   ROLES    AGE    VERSION
node-0   Ready    <none>   10m   v1.32.3
node-1   Ready    <none>   10m   v1.32.3
```

Next: [Provisioning Pod Network Routes](11-pod-network-routes.md)