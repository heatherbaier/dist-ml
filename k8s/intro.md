# Introduction to Kubernetes

## What is K8S, and why use it over Torque / Batch systems?

Here at William & Mary, we have access to two types of clusters - a Torque based batch cluster (see (this)[https://hmbaier.gitbook.io/distributed-ml-w-and-m/the-batch-system/what-is-the-batch-system)] guide), and a Kubernetes cluster.  Batch clusters operate in a fundamentally different way than Kubernetes clusters.  The key distinctions at William and Mary are:

**Node Architectures & Resource Requests**
In our batch systems, all nodes within a cluster are identical.  In our K8S clusters, you have mixed architectures (i.e., one node may have 0 GPUs, another may have 12).  This has implications for how resources are requested.  In our Batch system, if you wanted to check out a node with 12 cores and 128GB of memory, you could look up what node types have the memory you need (in this example, assuming "vortex" nodes have 128GB of memory total), and then explicitly specify what cores you want:
```
#!/bin/tcsh
#PBS -N demojob
#PBS -l nodes=1:vortex:ppn=12
#PBS -l walltime=00:30:00

echo "Hello world!"
```

In kubernetes, you don't specify the explicit node or node type you want - instead you request resources, and the K8S scheduler automatically identifies the optimal node for you to run on.  A resource request looks something like:
```yaml
  containers:
    - name: pytorch-setup-container
      image: "nvidia/samples:vectoradd-cuda11.2.1"
      resources:
        requests:
          memory: "32Gi"
          nvidia.com/gpu: 1
        limits:
          memory: "32Gi"
          nvidia.com/gpu: 1
```

## High level "what's going on here"
