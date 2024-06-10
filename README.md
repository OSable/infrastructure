# OSable's Kubernetes cluster
This repository contains the [ansible playbooks](/ansible/) to install a k3s cluster with Calico CNI using the eBPF dataplane and FluxCD. \
It also contains all of the configuration for our current cluster, secrets excluded. 

The contents of this repository are based off of [this document](k3s-cluster.md) I wrote to manually create a cluster using K3s, Calico using the eBPF dataplane, MetalLB, cert-manager and Traefik. 
