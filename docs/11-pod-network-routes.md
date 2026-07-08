# Provisioning Pod Network Routes

Pods scheduled to a node receive an IP address from the node's Pod CIDR range. At this point pods can not communicate with other pods running on different nodes due to missing network [routes](https://cloud.google.com/compute/docs/vpc/routes).

In this lab you will create a route for each worker node that maps the node's Pod CIDR range to the node's internal IP address.

> There are [other ways](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this) to implement the Kubernetes networking model.

## The Routing Table

In this section you will gather the information required to create routes in the `kubernetes-the-hard-way` VPC network.

Print the internal IP address and Pod CIDR range for each worker instance:

```bash
{
  SERVER_IP=$(grep server machines.txt | cut -d " " -f 1)
  NODE_0_IP=$(grep node-0 machines.txt | cut -d " " -f 1)
  NODE_0_SUBNET=$(grep node-0 machines.txt | cut -d " " -f 4)
  NODE_1_IP=$(grep node-1 machines.txt | cut -d " " -f 1)
  NODE_1_SUBNET=$(grep node-1 machines.txt | cut -d " " -f 4)
}
```

**Why:** This command block extracts the IP addresses and pod CIDR subnets for each machine from the `machines.txt` file and stores them in environment variables . This data extraction is necessary because the subsequent routing commands need these specific values, and storing them as variables reduces errors and avoids manually typing IP addresses multiple times.

```bash
ssh root@server <<EOF
  ip route add ${NODE_0_SUBNET} via ${NODE_0_IP}
  ip route add ${NODE_1_SUBNET} via ${NODE_1_IP}
EOF
```

**Why:** This command adds static routes on the server machine so it can route traffic to pods on both worker nodes by directing packets destined for each node's Pod CIDR range to that node's IP address . This is necessary because the server runs the API server and other control plane components that need to communicate with pods running on worker nodes for log retrieval, exec commands, and service routing. Without these routes, the server cannot reach pods on the worker nodes.

```bash
ssh root@node-0 <<EOF
  ip route add ${NODE_1_SUBNET} via ${NODE_1_IP}
EOF
```

**Why:** This adds a route on `node-0` that directs traffic destined for `node-1`'s Pod CIDR range through `node-1`'s internal IP address. This enables pods on `node-0` to communicate with pods on `node-1`, which is essential for the Kubernetes pod-to-pod networking model where any pod should be able to reach any other pod regardless of which node they're scheduled on.

```bash
ssh root@node-1 <<EOF
  ip route add ${NODE_0_SUBNET} via ${NODE_0_IP}
EOF
```

**Why:** This adds a route on `node-1` that directs traffic destined for `node-0`'s Pod CIDR range through `node-0`'s internal IP address. This completes the bidirectional routing setup, allowing pods on `node-1` to reach pods on `node-0`. Both worker nodes now have routes to each other's pod ranges, fulfilling the Kubernetes networking requirement that all pods can communicate without NAT.

## Verification 

```bash
ssh root@server ip route
```

**Why:** This displays the routing table on the server to verify that the routes to both worker node pod subnets were successfully added. This confirmation ensures the server can reach pods on both worker nodes before proceeding with the smoke test.

```text
default via XXX.XXX.XXX.XXX dev ens160 
10.200.0.0/24 via XXX.XXX.XXX.XXX dev ens160 
10.200.1.0/24 via XXX.XXX.XXX.XXX dev ens160 
XXX.XXX.XXX.0/24 dev ens160 proto kernel scope link src XXX.XXX.XXX.XXX 
```

```bash
ssh root@node-0 ip route
```

**Why:** This displays the routing table on `node-0` to verify that the route to `node-1`'s pod subnet was successfully added. This ensures `node-0` can route traffic to pods running on `node-1`.

```text
default via XXX.XXX.XXX.XXX dev ens160 
10.200.1.0/24 via XXX.XXX.XXX.XXX dev ens160 
XXX.XXX.XXX.0/24 dev ens160 proto kernel scope link src XXX.XXX.XXX.XXX 
```

```bash
ssh root@node-1 ip route
```

**Why:** This displays the routing table on `node-1` to verify that the route to `node-0`'s pod subnet was successfully added. This confirms `node-1` can route traffic to pods running on `node-0`, completing the bidirectional routing verification.

```text
default via XXX.XXX.XXX.XXX dev ens160 
10.200.0.0/24 via XXX.XXX.XXX.XXX dev ens160 
XXX.XXX.XXX.0/24 dev ens160 proto kernel scope link src XXX.XXX.XXX.XXX 
```

Next: [Smoke Test](12-smoke-test.md)