# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the machines required for setting up a Kubernetes cluster.

## Machine Database

This tutorial will leverage a text file, which will serve as a machine database, to store the various machine attributes that will be used when setting up the Kubernetes control plane and worker nodes. The following schema represents entries in the machine database, one entry per line:

```text
IPV4_ADDRESS FQDN HOSTNAME POD_SUBNET
```

Each of the columns corresponds to a machine IP address `IPV4_ADDRESS`, fully qualified domain name `FQDN`, host name `HOSTNAME`, and the IP subnet `POD_SUBNET`. Kubernetes assigns one IP address per `pod` and the `POD_SUBNET` represents the unique IP address range assigned to each machine in the cluster for doing so.

Here is an example machine database similar to the one used when creating this tutorial. Notice the IP addresses have been masked out. Your machines can be assigned any IP address as long as each machine is reachable from each other and the `jumpbox`.

```bash
cat machines.txt
```

**Why:** This displays the contents of the `machines.txt` file to verify it contains the expected entries before proceeding. At this point in the tutorial, you should have created this file manually with the IP addresses, FQDNs, hostnames, and pod subnets for your three machines.

```text
XXX.XXX.XXX.XXX server.kubernetes.local server
XXX.XXX.XXX.XXX node-0.kubernetes.local node-0 10.200.0.0/24
XXX.XXX.XXX.XXX node-1.kubernetes.local node-1 10.200.1.0/24
```

Now it's your turn to create a `machines.txt` file with the details for the three machines you will be using to create your Kubernetes cluster. Use the example machine database from above and add the details for your machines.

## Configuring SSH Access

SSH will be used to configure the machines in the cluster. Verify that you have `root` SSH access to each machine listed in your machine database. You may need to enable root SSH access on each node by updating the sshd_config file and restarting the SSH server.

### Enable root SSH Access

If `root` SSH access is enabled for each of your machines you can skip this section.

By default, a new `debian` install disables SSH access for the `root` user. This is done for security reasons as the `root` user has total administrative control of unix-like systems. If a weak password is used on a machine connected to the internet, well, let's just say it's only a matter of time before your machine belongs to someone else. As mentioned earlier, we are going to enable `root` access over SSH in order to streamline the steps in this tutorial. Security is a tradeoff, and in this case, we are optimizing for convenience. Log on to each machine via SSH using your user account, then switch to the `root` user using the `su` command:

```bash
su - root
```

**Why:** This switches to the root user after logging in with a regular user account, providing the administrative privileges needed to modify system-wide SSH configuration files. This step is necessary because the SSH daemon configuration file (`/etc/ssh/sshd_config`) is only writable by root, and subsequent commands in this section require root permissions to enable SSH access for the root user.

Edit the `/etc/ssh/sshd_config` SSH daemon configuration file and set the `PermitRootLogin` option to `yes`:

```bash
sed -i \
  's/^#*PermitRootLogin.*/PermitRootLogin yes/' \
  /etc/ssh/sshd_config
```

**Why:** This modifies the SSH daemon configuration to explicitly allow root login, replacing any existing `PermitRootLogin` line (commented or uncommented) with `PermitRootLogin yes`. This step is necessary because Debian and its derivatives disable root SSH access by default, but this tutorial requires root access to streamline the provisioning of multiple machines without repeatedly using `sudo` or switching users.

Restart the `sshd` SSH server to pick up the updated configuration file:

```bash
systemctl restart sshd
```

**Why:** This restarts the SSH daemon to apply the new `PermitRootLogin` configuration, making the change effective immediately without requiring a system reboot. Without this restart, the SSH server would continue using the old configuration and root login would remain disabled.

### Generate and Distribute SSH Keys

In this section you will generate and distribute an SSH keypair to the `server`, `node-0`, and `node-1`, machines, which will be used to run commands on those machines throughout this tutorial. Run the following commands from the `jumpbox` machine.

Generate a new SSH key:

```bash
ssh-keygen
```

**Why:** This generates a new SSH public/private key pair on the jumpbox machine, establishing a cryptographic identity for passwordless authentication. This step eliminates the need to enter passwords repeatedly when running commands across all cluster machines throughout the tutorial, which would otherwise become tedious and error-prone given the frequency of SSH connections required.

```text
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /root/.ssh/id_rsa
Your public key has been saved in /root/.ssh/id_rsa.pub
```

Copy the SSH public key to each machine:

```bash
while read IP FQDN HOST SUBNET; do
  ssh-copy-id root@${IP}
done < machines.txt
```

**Why:** This loop reads each machine's entry from `machines.txt` and copies the jumpbox's public SSH key to that machine's `~/.ssh/authorized_keys` file, authenticating with the password for each target. This step is necessary to establish trust so subsequent SSH connections from the jumpbox to each cluster machine will authenticate automatically using the private key, enabling passwordless automation of the many remote commands that follow in the tutorial.

Once each key is added, verify SSH public key access is working:

```bash
while read IP FQDN HOST SUBNET; do
  ssh -n root@${IP} hostname
done < machines.txt
```

**Why:** This loop tests that SSH key-based authentication works correctly by connecting to each machine and running `hostname` non-interactively (using the `-n` flag) to verify each machine responds with its hostname. This is a critical validation step that ensures all machines are properly configured for passwordless SSH before proceeding—without this, later automated commands would fail, making the tutorial difficult to complete.

```text
server
node-0
node-1
```

## Hostnames

In this section you will assign hostnames to the `server`, `node-0`, and `node-1` machines. The hostname will be used when executing commands from the `jumpbox` to each machine. The hostname also plays a major role within the cluster. Instead of Kubernetes clients using an IP address to issue commands to the Kubernetes API server, those clients will use the `server` hostname instead. Hostnames are also used by each worker machine, `node-0` and `node-1` when registering with a given Kubernetes cluster.

To configure the hostname for each machine, run the following commands on the `jumpbox`.

Set the hostname on each machine listed in the `machines.txt` file:

```bash
while read IP FQDN HOST SUBNET; do
    CMD="sed -i 's/^127.0.1.1.*/127.0.1.1\t${FQDN} ${HOST}/' /etc/hosts"
    ssh -n root@${IP} "$CMD"
    ssh -n root@${IP} hostnamectl set-hostname ${HOST}
    ssh -n root@${IP} systemctl restart systemd-hostnamed
done < machines.txt
```

**Why:** This loop configures the hostname on each cluster machine by performing three separate actions per iteration: first updating `/etc/hosts` with the machine's FQDN and short hostname (which ensures the loopback address resolves to the correct hostname), then setting the system hostname using `hostnamectl` (which configures the system's transient, static, and pretty hostnames), and finally restarting `systemd-hostnamed` to apply the changes immediately. This is necessary because Kubernetes clients rely on consistent, predictable hostname resolution both internally and when connecting to the API server, and the hostname is also used by worker nodes when registering with the control plane.

Verify the hostname is set on each machine:

```bash
while read IP FQDN HOST SUBNET; do
  ssh -n root@${IP} hostname --fqdn
done < machines.txt
```

**Why:** This loop retrieves and displays the fully qualified domain name from each machine to confirm the hostname changes were applied correctly. This verification is essential because an incorrectly set hostname would break internal Kubernetes communication and cause clients to fail when addressing the API server by its FQDN.

```text
server.kubernetes.local
node-0.kubernetes.local
node-1.kubernetes.local
```

## Host Lookup Table

In this section you will generate a `hosts` file which will be appended to `/etc/hosts` file on the `jumpbox` and to the `/etc/hosts` files on all three cluster members used for this tutorial. This will allow each machine to be reachable using a hostname such as `server`, `node-0`, or `node-1`.

Create a new `hosts` file and add a header to identify the machines being added:

```bash
echo "" > hosts
```

**Why:** This creates an empty `hosts` file in the current directory (or overwrites any existing file), preparing a clean starting point for the host entries. This step is necessary because we're building the file from scratch and want to avoid any stale entries from previous runs.

```bash
echo "# Kubernetes The Hard Way" >> hosts
```

**Why:** This appends a descriptive comment header to the `hosts` file to identify its purpose and origin, which helps distinguish these entries from other content when the file is later appended to `/etc/hosts`. This annotation is useful for debugging and maintenance since `/etc/hosts` typically contains multiple sets of entries from different sources.

Generate a host entry for each machine in the `machines.txt` file and append it to the `hosts` file:

```bash
while read IP FQDN HOST SUBNET; do
    ENTRY="${IP} ${FQDN} ${HOST}"
    echo $ENTRY >> hosts
done < machines.txt
```

**Why:** This loop reads each machine's attributes from `machines.txt` and creates host entry lines that map each machine's IP address to both its fully qualified domain name and its short hostname. This is necessary because having both forms defined ensures that tools and commands can resolve either the FQDN or the short name to the correct IP address, which is critical for consistent machine discovery within the cluster.

Review the host entries in the `hosts` file:

```bash
cat hosts
```

**Why:** This displays the generated `hosts` file to verify the entries were created correctly before distributing them to all machines. Reviewing the file now catches any mis-formatting or missing entries that would otherwise cause hostname resolution failures throughout the cluster.

```text

# Kubernetes The Hard Way
XXX.XXX.XXX.XXX server.kubernetes.local server
XXX.XXX.XXX.XXX node-0.kubernetes.local node-0
XXX.XXX.XXX.XXX node-1.kubernetes.local node-1
```

## Adding `/etc/hosts` Entries To A Local Machine

In this section you will append the DNS entries from the `hosts` file to the local `/etc/hosts` file on your `jumpbox` machine.

Append the DNS entries from `hosts` to `/etc/hosts`:

```bash
cat hosts >> /etc/hosts
```

**Why:** This appends the generated host entries to the jumpbox's `/etc/hosts` file, enabling the jumpbox to resolve the cluster machines' hostnames locally without requiring a DNS server. This is necessary because the jumpbox needs to communicate with all cluster machines by name throughout the tutorial, and without local resolution, you'd need to use IP addresses manually for every command.

Verify that the `/etc/hosts` file has been updated:

```bash
cat /etc/hosts
```

**Why:** This displays the complete `/etc/hosts` file to confirm the Kubernetes host entries were successfully appended alongside the existing system entries. Verifying this ensures the jumpbox can now resolve all cluster machine hostnames, which is essential before testing connectivity with the next command.

```text
127.0.0.1       localhost
127.0.1.1       jumpbox

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

# Kubernetes The Hard Way
XXX.XXX.XXX.XXX server.kubernetes.local server
XXX.XXX.XXX.XXX node-0.kubernetes.local node-0
XXX.XXX.XXX.XXX node-1.kubernetes.local node-1
```

At this point you should be able to SSH to each machine listed in the `machines.txt` file using a hostname.

```bash
for host in server node-0 node-1
   do ssh root@${host} hostname
done
```

**Why:** This loop tests the newly added host entries by connecting to each cluster machine using its short hostname and running `hostname` to verify the connection works. This step is crucial because it validates that both the hostname resolution and SSH key authentication are working correctly before proceeding to more complex operations, and ensures the jumpbox can reach all machines by their assigned names.

```text
server
node-0
node-1
```

## Adding `/etc/hosts` Entries To The Remote Machines

In this section you will append the host entries from `hosts` to `/etc/hosts` on each machine listed in the `machines.txt` text file.

Copy the `hosts` file to each machine and append the contents to `/etc/hosts`:

```bash
while read IP FQDN HOST SUBNET; do
  scp hosts root@${HOST}:~/
  ssh -n \
    root@${HOST} "cat hosts >> /etc/hosts"
done < machines.txt
```

**Why:** This loop distributes the host entries to each cluster machine by first copying the `hosts` file from the jumpbox to the remote machine's home directory using SCP, then appending the file's contents to the remote machine's `/etc/hosts` file. This is necessary because each machine in the cluster needs to resolve every other machine's hostname internally—Kubernetes components and workloads running on one node need to communicate with API server endpoints and other services on different nodes, and this local resolution ensures reliability even if the network's DNS infrastructure experiences issues.

At this point, hostnames can be used when connecting to machines from your `jumpbox` machine, or any of the three machines in the Kubernetes cluster. Instead of using IP addresses you can now connect to machines using a hostname such as `server`, `node-0`, or `node-1`.

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)