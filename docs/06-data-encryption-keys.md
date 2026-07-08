# Generating the Data Encryption Config and Key

Kubernetes stores a variety of data including cluster state, application configurations, and secrets. Kubernetes supports the ability to [encrypt](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data) cluster data at rest.

In this lab you will generate an encryption key and an [encryption config](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#understanding-the-encryption-at-rest-configuration) suitable for encrypting Kubernetes Secrets.

## The Encryption Key

Generate an encryption key:

```bash
export ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

**Why:** This generates a cryptographically secure 32-byte (256-bit) random key, encoded in base64, which will be used as the encryption secret for AES-CBC encryption of Kubernetes Secrets stored in etcd. The key is generated from `/dev/urandom`, which provides high-quality random data suitable for cryptographic use, and storing it in an environment variable allows the subsequent `envsubst` command to inject it into the configuration file without hardcoding the value.

## The Encryption Config File

Create the `encryption-config.yaml` encryption config file:

```bash
envsubst < configs/encryption-config.yaml \
  > encryption-config.yaml
```

**Why:** This command processes the `encryption-config.yaml` template file from the `configs/` directory, substituting the `ENCRYPTION_KEY` environment variable into the placeholder, and writes the rendered configuration to the current directory. The resulting file defines that Kubernetes Secrets should be encrypted using the AES-CBC provider with the generated key, with `identity: {}` as a fallback provider to allow reading unencrypted data during the migration to encryption. This configuration is necessary because the API server must be explicitly told to encrypt data at rest; otherwise it defaults to storing Secrets as plain text in etcd.

Copy the `encryption-config.yaml` encryption config file to each controller instance:

```bash
scp encryption-config.yaml root@server:~/
```

**Why:** This copies the encryption configuration file to the control plane server's home directory, where it will be moved to `/var/lib/kubernetes/` and referenced by the API server via the `--encryption-provider-config` flag. The file contains the encryption key material, so it must be placed on the control plane node where the API server runs—the worker nodes do not need this file because encryption and decryption of Secrets happen entirely within the API server when reading from and writing to etcd.

Next: [Bootstrapping the etcd Cluster](07-bootstrapping-etcd.md)