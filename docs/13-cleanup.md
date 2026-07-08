# Cleaning Up

In this lab you will delete the compute resources created during this tutorial.

## Compute Instances

Previous versions of this guide made use of GCP resources for various aspects of compute and networking. The current version is agnostic, and all configuration is performed on the `jumpbox`, `server`, or nodes.

Clean up is as simple as deleting all virtual machines you created for this exercise.

**Why:** This step is necessary to avoid ongoing costs and to free up resources on your infrastructure provider. Since this tutorial is platform-agnostic and no longer uses cloud-specific APIs for resource management, the cleanup process is straightforward—you simply delete the virtual machines (jumpbox, server, node-0, node-1) that you provisioned at the start. There are no additional cloud resources (like VPC routes, firewall rules, or load balancers) to manage because this version of the tutorial configures networking manually within the VMs rather than through cloud provider integration.

Next: [Start Over](../README.md)