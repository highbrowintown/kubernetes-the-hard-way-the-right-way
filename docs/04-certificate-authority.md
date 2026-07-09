# Provisioning a CA and Generating TLS Certificates

In this lab you will provision a [PKI Infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure) using openssl to bootstrap a Certificate Authority, and generate TLS certificates for the following components: kube-apiserver, kube-controller-manager, kube-scheduler, kubelet, and kube-proxy. The commands in this section should be run from the `jumpbox`.

## Certificate Authority

In this section you will provision a Certificate Authority that can be used to generate additional TLS certificates for the other Kubernetes components. Setting up CA and generating certificates using `openssl` can be time-consuming, especially when doing it for the first time. To streamline this lab, I've included an openssl configuration file `ca.conf`, which defines all the details needed to generate certificates for each Kubernetes component.

Take a moment to review the `ca.conf` configuration file:

```bash
cat ca.conf
```

**Why:** This displays the contents of the `ca.conf` OpenSSL configuration file so you can see the certificate profiles, key usage extensions, and subject alternative name definitions that will be used for each Kubernetes component. Reviewing this file helps you understand what settings are being applied when generating certificates, which is important for troubleshooting and for learning how OpenSSL configurations work in practice.

You don't need to understand everything in the `ca.conf` file to complete this tutorial, but you should consider it a starting point for learning `openssl` and the configuration that goes into managing certificates at a high level.

Every certificate authority starts with a private key and root certificate. In this section we are going to create a self-signed certificate authority, and while that's all we need for this tutorial, this shouldn't be considered something you would do in a real-world production environment.

Generate the CA configuration file, certificate, and private key:

```text
openssl genrsa -out ca.key 4096
```

**Why:** Generates the Certificate Authority's private key using RSA at 4096 bits. Everything else in the cluster's trust chain ultimately depends on this one key staying private.

```bash
openssl req -x509 -new -sha512 -noenc -key ca.key -days 3653 -config ca.conf -out ca.crt
```

**Why:** Creates a self-signed X.509 root certificate from that key, valid for 3653 days (~10 years). `-x509` tells `openssl req` to output a self-signed certificate directly instead of a CSR, since a root CA has no one else to sign it — it signs itself. `-noenc` skips encrypting the private key with a passphrase, so it can be used non-interactively throughout the rest of this tutorial. This certificate becomes the root of trust every other certificate generated later will be signed against.

Results:

```bash
ca.crt ca.key
```

## Create Client and Server Certificates

In this section you will generate client and server certificates for each Kubernetes component and a client certificate for the Kubernetes `admin` user.

Generate the certificates and private keys:

```bash
certs=(
  "admin" "node-0" "node-1"
  "kube-proxy" "kube-scheduler"
  "kube-controller-manager"
  "kube-api-server"
  "service-accounts"
)
```

**Why:** This defines an array of certificate names that will be generated for each Kubernetes component and user. Each entry corresponds to a different principal that needs to authenticate with the Kubernetes API - the admin user for cluster management, worker nodes for kubelet authentication, service components like kube-proxy and the controller managers, the API server itself for serving TLS, and a special certificate for signing service account tokens .

```bash
for i in ${certs[*]}; do
  openssl genrsa -out "${i}.key" 4096

  openssl req -new -key "${i}.key" -sha256 -config "ca.conf" -section ${i} -out "${i}.csr"

  openssl x509 -req -days 3653 -in "${i}.csr" -copy_extensions copyall -sha256 -CA "ca.crt" -CAkey "ca.key" -CAcreateserial -out "${i}.crt"
done
```

**Why:** This loop generates a complete certificate pair for each Kubernetes component listed in the `certs` array. For each component, it creates a 4096-bit RSA private key, generates a certificate signing request using the appropriate section from `ca.conf`, and then signs the CSR with the CA certificate to produce a valid TLS certificate valid for 10 years . The `-copy_extensions copyall` flag ensures that important X.509 extensions like subjectAltNames are preserved from the CSR to the final certificate, which is critical for certificates like `kube-api-server` that need to include DNS names and IP addresses for client validation .

The results of running the above command will generate a private key, certificate request, and signed SSL certificate for each of the Kubernetes components. You can list the generated files with the following command:

```bash
ls -1 *.crt *.key *.csr
```

**Why:** This lists all the certificate, private key, and CSR files that were generated by the loop, allowing you to verify that all expected component certificates were created successfully. This confirmation is important before distributing certificates to the cluster machines, as missing certificates would cause authentication failures when Kubernetes components attempt to start up.

## Distribute the Client and Server Certificates

In this section you will copy the various certificates to every machine at a path where each Kubernetes component will search for its certificate pair. In a real-world environment these certificates should be treated like a set of sensitive secrets as they are used as credentials by the Kubernetes components to authenticate to each other.

Copy the appropriate certificates and private keys to the `node-0` and `node-1` machines:

```bash
for host in node-0 node-1; do
  ssh root@${host} mkdir /var/lib/kubelet/

  scp ca.crt root@${host}:/var/lib/kubelet/

  scp ${host}.crt root@${host}:/var/lib/kubelet/kubelet.crt

  scp ${host}.key root@${host}:/var/lib/kubelet/kubelet.key
done
```

**Why:** This loop copies the necessary certificates to each worker node for the kubelet service. It creates the `/var/lib/kubelet/` directory where the kubelet expects to find its certificates, copies the CA certificate so the kubelet can validate the API server's certificate, and then copies the node-specific certificate and private key to standard filenames (`kubelet.crt` and `kubelet.key`) that the kubelet service expects . Each node receives only its own certificate, not the certificates for other nodes, maintaining proper separation of credentials.

Copy the appropriate certificates and private keys to the `server` machine:

```bash
scp ca.key ca.crt kube-api-server.key kube-api-server.crt service-accounts.key service-accounts.crt root@server:~/
```

**Why:** This copies the CA private key, CA certificate, API server certificate pair, and service account certificate pair to the `server` machine's home directory. The API server needs its own certificate to serve TLS-encrypted HTTPS traffic to clients, and the CA certificate to validate client certificates presented by kubelets and other components — validating a certificate only ever requires the CA's public certificate, not its private key. The CA private key is copied here because `kube-controller-manager`, which also runs on this machine, uses it (via `--cluster-signing-cert-file` and `--cluster-signing-key-file`) to sign new certificates when the CertificateSigningRequest API is used, not because the API server itself needs it. The service account certificates are placed on the server because `kube-controller-manager` uses the service account key pair to generate and sign service account tokens for pods.

> The `kube-proxy`, `kube-controller-manager`, `kube-scheduler`, and `kubelet` client certificates will be used to generate client authentication configuration files in the next lab.

Next: [Generating Kubernetes Configuration Files for Authentication](05-kubernetes-configuration-files.md)