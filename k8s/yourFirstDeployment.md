# Your First K8S Deployment
In kubernetes, there are a number of different ways to submit code to the system.  The two primary techniques you will be using are through "Pods" and "Jobs".

## What is a Pod?
Deploying a single pod is the simplest way to run code against a K8S cluster.  Pods are made up of a few elements, including at least (a) your resource request; (b) your image; and (c) the code you would like to run.  First we'll go through the high level description of each element of a pod, and then we'll show an actual submission script example.
> One critical consideration about Pods is that they are not persistent - anything you do within them will be deleted when the pod is shut down.  We will cover how to achieve persistence in a later tutorial.

### Resource Requests
A single pod can request resources up to what is available on the largest individual node on a cluster, but Pods cannot expand beyond a single physical, available computer.  For example, if you have - somewhere in your cluster - a machine with 1 terrabyte of memory and 64 CPUs, then a pod can be built with up to that specification (but, you may have to wait in a queue to get access to those resources if others are using them).  Alternatively, if you have one node with 12 GPUs, then either 12 users could all request 1 GPU in a pod, or one user could request 12 and be allocated to the same node (though, again, you may end up in a queue if resources are not available at any given time).

One unique feature of kubernetes is that you can request both a *minimum* amount of resources, and a *limit*.  Without the limit, jobs can scale up or down according to demand, but for now we will always be specifying both - ensuring that you receive exactly the resources you want, no more and no less.

### Your Pod's Image
Pods are based on images - when you run a pod, you must also specify the base image that will run within it.  Think about a pod as an individual computer you control, and the image is the operating system that you would like it to run.  It's important that the operating system has any drivers you might require (i.e., NVIDIA drivers for GPUs), and you may want it to have some software you frequently use so you don't have to reinstall it every time (i.e., Python).  You can choose the image you want from a menu, which we'll talk about in a later tutorial.

### Your code
Just like any other system, once you've checked out your compute resources, you then need to specify the actual code you want to run.  This isn't all that different from logging into a computer and typing, for example, "python myscript.py" to execute a program.

## Define your First Kubernetes Pod
In Kubernetes, all submissons are called "deployments", reflecting the long history of kubernetes as a dynamic scheduler for website resources.  You can choose to deploy multiple things - for example, here we'll be deploying a single Pod, but later we'll deploy Jobs (which are very similar to jobs in a batch system).  To do this, first you'll need to log in to kubernetes.  Today, you'll require special permission to do so, and can access the kubernetes frontend by jumping from Bora to the K8S frontend "cm" (ssh <your_username>@bora.sciclone.wm.edu, and then once connected to Bora, ssh again to <your_username>@cm.geo.sciclone.wm.edu).  

Once you're logged in, you'll be in your Kubernetes home directory (/home/<your_username>).  Here, you can create job files to submit to the K8S cluster.

Let's go ahead and specify our first job file - a *yaml* which defines what we want to do.  In this example, we'll request a single pod with one cpu, and then walk through some basic commands and how to observe what's happening in the Pod.  Using nano or vim, create a file with the following code:

myFirstPod.yml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-info-pod
spec:
  containers:
    - name: resource-info-container
      image: "nvidia/samples:vectoradd-cuda11.2.1"
      resources:
        requests:
          memory: "1Gi"
          cpu: "1"
        limits:
          memory: "1Gi"
          cpu: "1"
      command: ["/bin/sh"]
      args:
        - "-c"
        - |
          echo "CPU and Memory limits set for this container:"
          echo "Memory limit: $MEMORY_LIMIT"
          echo "CPU limit: $CPU_LIMIT"
          echo "Running the main container process indefinitely..."
          sleep 1800
      env:
        - name: MEMORY_LIMIT
          valueFrom:
            resourceFieldRef:
              containerName: resource-info-container
              resource: limits.memory
        - name: CPU_LIMIT
          valueFrom:
            resourceFieldRef:
              containerName: resource-info-container
              resource: limits.cpu
```

The above pod does the following:
- It defines the kind of resource we want to deploy - in this case, a Pod.
- It names the pod - this can be anything you want, sans a few special characters, and is the name you'll use to inspect the pod later.
- It specifies an activeDeadlineSeconds - this is the walltime, at which point the pod will be destroyed.
- It defines the base image we want to use, which in this example is a nvidia image with various elements required for GPU utilization.
- It asks for 1 gigabyte of memory, and one cpu.  We request both a minimum (resources) and a maximum (limites).
- I defines the commands to run in the pod.  Here, we just ask it to print the CPU and Memory vailable.
- Finally, it defines a few environmental variables that will be available inside the pod.  These define the two we use - MEMORY_LIMIT and CPU_LIMIT, but you can use this method to ascribe values ot any environmental variables you want.

## Deploying your Pod
Deploying pods in kubernetes is very simple.  In this case, you would run:
```bash
kubectl apply -f myFirstPod.yml
```

If succesful, you will receive a message indicating that "pod/resource-info-pod" was created.  You can confirm the status of your pods by typing:
```bash
kubectl get pods
```

Which will show you the status of all of your running pods.  You should see something like this:
![image](https://github.com/heatherbaier/dist-ml/assets/7882645/45d01c6f-2f1a-45d6-8014-c5c566cfcbb5)

## Reading the outputs from your Pod
To see the logs generated by your pod, you can run:
```bash
kubectl logs <name-of-pod>
```
If you run kubectl logs resource-info-container, you will see an output similar to this:
![image](https://github.com/heatherbaier/dist-ml/assets/7882645/bea85702-6d72-42e1-8b02-cf42581011fd)

> Note that logs are not persistent, and are deleted when the pod is deleted.

## Logging in to your Pod
You can log in to any pod to debug any running process, install software, or just poke around.  Note that your pod has to be running for this to work - the sleep infinity at the end of the pod is what allows this to happen in this example, which will keep the pod alive until it's walltime (defined in activeDeadlineSeconds) arrives - 30 minutes in this example.  To log into the pod:
```
kubectl exec -it resource-info-pod -- /bin/bash
```

Once you execute that line, it will drop you into a new terminal that is running inside of the pod. To confirm you're inside the pod, you can run a command such as:
```bash
cat /etc/os-release
```
This will output the base image the pod is using, which in this example is:
![image](https://github.com/heatherbaier/dist-ml/assets/7882645/c8453e79-9910-4842-a5b3-b0d2c1b5797d)

Once you're done, you can type "exit".

> As you type "kubectl" a lot, you may want to bind it to something shorter.  To do so, go into your home directory and type in "nano .bashrc".  If you want to bind "kubectl" to just "k", you can add the line `alias k='kubectl'` to that file.  Once you save it and restart your shell, you can replace "kubectl" with just "k" from that moment onward.

## Deleting your Pod
Today, if you do not delete your pod, it will exist until all commands have run or the "activeDeadlineSeconds" limit has bene reached.  In this example, that is until 30 minutes have passed - the script sleeps for an infinite time, so will not ever finish, thus resulting in the walltime trigger of 30 minutes being the event that will destroy the pod.  If you do not put "sleep" at the end of your commands, the pod will simply run and then complete, which we'll show in future tutorials.

If you want to manually delete a pod, you can always do so by typing:
```
kubectl delete pod <name-of-pod>
```

In this example, that would be kubectl delete pod resource-info-pod.  If succesful, that command will result in the output:
![image](https://github.com/heatherbaier/dist-ml/assets/7882645/b0f65487-0b19-47ec-8676-10ed1a4a660b)


## Gotchas and Notes
- Pods must have unique names.  You will receive an error if you try to launch two different pods with the same name.
- Remember, your pod is a completely independent OS from the frontend node.  Things you do on the frontend will generally not impact your pods, except in the YML files.
