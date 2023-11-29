# Multi-Pod, Multi-GPU with Torch
Here, we'll be requested 3 GPUs and running a torch classification on CIFAR using them.  In this case, it will be fully distributed: we don't care what node the pods are running on.

# Kubernetes Jobs
To distribute pods, we're going to use a new type of Kubernetes API: Jobs.  Jobs allow you to specify a number of pods that you want to spin up, and characteristics of each of them. For example:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: dist-torch
spec:
  parallelism: 2
  template:
    spec:
      restartPolicy: OnFailure
      volumes:
        - name: home-volume
          persistentVolumeClaim:
            claimName: dsmr-vol-01
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
          volumeMounts:
            - name: home-volume
              mountPath: /kube/home/
```

The above job will create two pods, each with at least 1 gpu.  The location of these pods may be arbitrary across the cluster, but each pod is guaranteed at least one gpu and 32 gigs of system memory.

> Note that when you submit a job, you will be able to see the pods it creates with kubectl get pods.  However, you will also be able to see the job using kubectl get jobs.  If you want to resubmit your job, you will need to delete the original job with kubectl delete job <job-name>.

# Defining the YAML file
Our YAML file for distribution will broadly look very similar to our single-pod jobs, but we'll define a parallelism of 4 (i.e., we'll be creating 4 pods).  Each pod will contain 1 gpu - you generally want individual pods to be as small as possible, as then you can fit more of them on more hardware.  For example, if we have 30 GPUs in the cluster total, but 12 of them are on one physical node and the rest are spread across 18 nodes with one GPU each, specifying your pod as having 1 GPU will allow you to run on any GPUs across the entire cluster.  If you were to specify 2 or more GPUs, you would only be able to run the pod on a machien with at least 2 GPUs, locking you out of the 18 single-GPU boxes.

There are a few other characteristics 

# Submitting the YAML file
Note that we won't be able to apply here - we need to *create* our job, as we're using the special "generateName" function.  Thus, you'll run something like: `k create -f 5_distributedTorch.yml`. 
