# Setting up Python and Conda in K8S
Depending on the base image you are using, the process to setup Conda and Python may be slightly different.  Here, we show how to do it using a common NVIDIA base image, but you may have to adapt some things to your own use case.

## File Persistence in Kubernetes
While individual pods are ephemeral (i.e., nothing written in them is saved), it is possible to have those pods write to locations that are persistent, and that multiple pods can access.  In kubernetes, these are referred to as *persistent volumes*.  You will be assigned a volume for your user.  In this example, we will create our conda environments inside a persistent volume, so that future pods can access our environments at later dates.

## Accessing a persistent volume
To access a persistent volume, you must specify both (a) the claimName for your volume, which the HPC team will provide, and (b) the path you want your persistent volume to be mounted in inside the pod.  In the below example, I have been asigned a 500GB volume with the claimName dsmr-vol-01.  You will not have access to this claim, and will need to replace it with your own.  I am mounting this persistent volume to the path /kube/home within my pod, and then printing out the total disk space available.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: claim-example
spec:
  restartPolicy: OnFailure
  activeDeadlineSeconds: 1800  # 30 minutes
  volumes:
    - name: home-volume
      persistentVolumeClaim:
        claimName: dsmr-vol-01
  containers:
    - name: conda-container
      image: "nvidia/samples:vectoradd-cuda11.2.1"
      volumeMounts:
        - name: home-volume
          mountPath: /kube/home/
      command: ["/bin/sh", "-c"]
      args:
        - |
          echo "Disk space usage for /kube/home volume:"
          df -h /kube/home
          echo "Sleeping indefinitely..."
          sleep infinity
```

Note that pods with claims will take a few seconds longer to spin up, as K8S must attach the appropriate disks to your pod.
