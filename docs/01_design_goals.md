# Design Goals

I would love to have a HA cluster with high fault tolerance and what not, but as a student, I have limited budget for this homelab project.
I will be keeping minimum expectations from this project for now.

> \[!NOTE]\
> Constraints due to less resources at my disposal, I plan to have just one control-plane node, but this is a single point of failure!

## Goals

- Learn more about monitoring and logging within the Kubernetes cluster.
- Host various projects (simple to complex) on the cluster and expose it to the internet.

## Resource Issues

The more PODs we run on the nodes, the more space is required. Same goes for the RAM. I have spent numerous hours waiting for PODs to come up on the Raspberry Pi 4 2GB RAM variant, so I guess it was not best to buy the base model! Make sure you have enough resources (cpu, memory, etc) when designing your cluster. I plan to convert the 2GB RAM Raspberry Pi 4 as the master node running the control-plane components, since they can be simple as they do very little except keeping the worker nodes in sync.

[Prev: Hardware](./00_hardware.md) | [Next: Installing the OS](./02_base_os.md)
