# Generating Kubernetes Configuration Files for Authentication

In this lab you will generate [Kubernetes client configuration files](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/), typically called kubeconfigs, which configure Kubernetes clients to connect and authenticate to Kubernetes API Servers.

## Client Authentication Configs

In this section you will generate kubeconfig files for the `kubelet` and the `admin` user.

### The kubelet Kubernetes Configuration File

When generating kubeconfig files for Kubelets the client certificate matching the Kubelet's node name must be used. This will ensure Kubelets are properly authorized by the Kubernetes [Node Authorizer](https://kubernetes.io/docs/reference/access-authn-authz/node/).

> The following commands must be run in the same directory used to generate the SSL certificates during the [Generating TLS Certificates](04-certificate-authority.md) lab.

Generate a kubeconfig file for the `node-0` and `node-1` worker nodes:

```bash
for host in node-0 node-1; do
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443 \
    --kubeconfig=${host}.kubeconfig

  kubectl config set-credentials system:node:${host} \
    --client-certificate=${host}.crt \
    --client-key=${host}.key \
    --embed-certs=true \
    --kubeconfig=${host}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${host} \
    --kubeconfig=${host}.kubeconfig

  kubectl config use-context default \
    --kubeconfig=${host}.kubeconfig
done
```

**Why:** This loop generates a kubeconfig file for each worker node's kubelet, establishing how the kubelet will authenticate to the API server. For each node, it defines a cluster endpoint pointing to the API server's HTTPS URL, sets the node's client certificate and key as credentials, and creates a context that ties these together . The `system:node:${host}` username is critical because the Kubernetes Node Authorizer specifically looks for this naming pattern to authorize kubelet API requests, and the certificate's Common Name must match exactly for the authorization to work properly .

Results:

```text
node-0.kubeconfig
node-1.kubeconfig
```

### The kube-proxy Kubernetes Configuration File

Generate a kubeconfig file for the `kube-proxy` service:

```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443 \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.crt \
    --client-key=kube-proxy.key \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config use-context default \
    --kubeconfig=kube-proxy.kubeconfig
}
```

**Why:** This generates a kubeconfig file for the kube-proxy service, which runs on every node and implements Kubernetes service networking via IPVS or iptables. The `system:kube-proxy` username corresponds to a ClusterRoleBinding that grants kube-proxy the necessary permissions to watch endpoint and service objects, and the certificate must be signed by the CA to enable mutual TLS authentication with the API server .

Results:

```text
kube-proxy.kubeconfig
```

### The kube-controller-manager Kubernetes Configuration File

Generate a kubeconfig file for the `kube-controller-manager` service:

```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.crt \
    --client-key=kube-controller-manager.key \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config use-context default \
    --kubeconfig=kube-controller-manager.kubeconfig
}
```

**Why:** This creates a kubeconfig file for the kube-controller-manager, which manages the suite of controllers that regulate cluster state including node lifecycle, replication, and service accounts. The `system:kube-controller-manager` user has high-level permissions defined in the `system:controller-manager` ClusterRole, and this file ensures the controller manager authenticates to the API server using its client certificate . The kubeconfig is stored on the control plane server because the controller manager runs there as a static pod or system service.

Results:

```text
kube-controller-manager.kubeconfig
```


### The kube-scheduler Kubernetes Configuration File

Generate a kubeconfig file for the `kube-scheduler` service:

```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.crt \
    --client-key=kube-scheduler.key \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config use-context default \
    --kubeconfig=kube-scheduler.kubeconfig
}
```

**Why:** This generates a kubeconfig file for the kube-scheduler, which watches newly created pods with no assigned node and selects a suitable node based on resource availability and scheduling policies. The `system:kube-scheduler` user has permissions to bind pods to nodes through its `system:scheduler` ClusterRole, and this configuration allows the scheduler to communicate with the API server using its client certificate for authentication . Like the controller manager, this file resides on the control plane server where the scheduler runs.

Results:

```text
kube-scheduler.kubeconfig
```

### The admin Kubernetes Configuration File

Generate a kubeconfig file for the `admin` user:

```bash
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.crt \
  --embed-certs=true \
  --server=https://server.kubernetes.local:6443 \
  --kubeconfig=kube-proxy.kubeconfig
```

**Why:** Defines the cluster entry inside `kube-proxy.kubeconfig` — the API server's address and the CA certificate needed to verify it. `--embed-certs=true` writes the actual certificate bytes into the kubeconfig file itself rather than a file path reference, so the resulting file is self-contained and portable to the node without needing `ca.crt` alongside it.

```bash
kubectl config set-credentials system:kube-proxy \
  --client-certificate=kube-proxy.crt \
  --client-key=kube-proxy.key \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig
```

**Why:** Adds kube-proxy's client certificate and key as a named credential. The `system:kube-proxy` identity corresponds to a ClusterRoleBinding that grants kube-proxy permission to watch Service and Endpoints objects, which it needs to build its iptables/IPVS rules.

```bash
kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig
```

**Why:** Ties the cluster and credential entries together into a named context so `kube-proxy.kubeconfig` knows which cluster to talk to using which identity.

```bash
kubectl config use-context default \
  --kubeconfig=kube-proxy.kubeconfig
```

**Why:** Sets this as the active context in the file, so kube-proxy doesn't need to specify `--context` explicitly when it starts up and reads this kubeconfig.

Results:

```text
admin.kubeconfig
```

## Distribute the Kubernetes Configuration Files

Copy the `kubelet` and `kube-proxy` kubeconfig files to the `node-0` and `node-1` machines:

```bash
for host in node-0 node-1; do
  ssh root@${host} "mkdir -p /var/lib/{kube-proxy,kubelet}"

  scp kube-proxy.kubeconfig \
    root@${host}:/var/lib/kube-proxy/kubeconfig

  scp ${host}.kubeconfig \
    root@${host}:/var/lib/kubelet/kubeconfig
done
```

**Why:** This loop creates the necessary directories on each worker node and copies the kubeconfig files to their expected locations. The kube-proxy kubeconfig goes to `/var/lib/kube-proxy/kubeconfig` because the kube-proxy systemd service or container reads it from this path, and the node-specific kubelet kubeconfig goes to `/var/lib/kubelet/kubeconfig` as this is the default location where the kubelet service expects its configuration file . Each node receives its own node-specific kubelet kubeconfig and shares the same kube-proxy kubeconfig for the networking proxy service.

Copy the `kube-controller-manager` and `kube-scheduler` kubeconfig files to the `server` machine:

```bash
scp admin.kubeconfig \
  kube-controller-manager.kubeconfig \
  kube-scheduler.kubeconfig \
  root@server:~/
```

**Why:** This copies the control plane component kubeconfig files to the `server` machine's home directory, along with the admin kubeconfig for cluster administration. They land here only temporarily — in the next lab (`08-bootstrapping-kubernetes-controllers.md`), `kube-controller-manager.kubeconfig` and `kube-scheduler.kubeconfig` get moved into `/var/lib/kubernetes/`, which is the path their systemd unit files actually point to. The admin kubeconfig is useful to have here too, since you can run `kubectl` directly from the server using it to verify the cluster is operational before moving on.

Next: [Generating the Data Encryption Config and Key](06-data-encryption-keys.md)