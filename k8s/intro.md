# Introduction to Kubernetes

## What is K8S, and why use it over Torque / Batch systems?

Here at William & Mary, we have access to two types of clusters - a Torque based batch cluster (see (this)[https://hmbaier.gitbook.io/distributed-ml-w-and-m/the-batch-system/what-is-the-batch-system)] guide), and a Kubernetes cluster.  Batch clusters operate in a fundamentally different way than Kubernetes clusters.  The key distinctions at William and Mary are:

##**Node Architectures & Resource Requests**

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
          cpu: "2"
        limits:
          memory: "32Gi"
          nvidia.com/gpu: 1
          cpu: "2"
```
Practically, this means you don't have to know much about the underlying architecture of the system - you just tell it what resources you need, and it will find them on the appropriate node(s).



**Pods vs. Running on Cores and Isolation**

In our the batch system, when you check out a series of cores, you are sharing some types of resources (i.e. memory) on an node.  Say, for example, you check out 6 cores on a node that has a total of 12, and then another user checks out the other 6.  Because our batch system doesn't have the ability to segment memory utilization, both users are now sharing the memory, and can crash one-anothers jobs if they use all of the memory on the node.  Because of that, we frequently suggest users check out all cores on a node.

K8S isolates your resources into a special construct called a "Pod".  Once a Pod is created on a node, it is allocated exactly the resources requested, and is guaranteed access to those resources, irrespective of what other user pods may be running on the node.  



**Containers and Research Replication**

One of the biggest advantages of Kubernetes is that jobs are run within dynamically built containers.  A container is comprised of two things: a base image (essentially, an operating system such as Ubuntu, RHEL, Cent, or Windows), and then anything installed on top of that operating system (i.e., a program you download).  When you request resources and a pod, you always specify the base image that the pod will load.  Say, for example, you always want a script that runs on Ubuntu 18.04 - you can build an image that will always be identical, irrespective of how external dependencies may change.  This is specified in your job files, and generally looks something like this:
```yaml
  containers:
    - name: pytorch-setup-container
      image: "nvidia/samples:vectoradd-cuda11.2.1"
```

The job submission file itself then specifies any commands you want to run within that operating system on the opd.  For example, after the base image example above is loaded (an nvidia debian environment), the below code would then install the program "wget" on every pod when the job is submitted.  You can choose to either bundle wget into the image itself, or install it later as a dependency, depending on your risk appetite for future changes to dependencies:
```yaml
     command:
        - /bin/bash
        - -c
        - |
          apt-get update && apt-get install -y wget 
```
