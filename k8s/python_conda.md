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
As a reminder, you would use:
kubectl apply -f persistence.yml

to deploy your pod.  You can then monitor it's progress being created with:
kubectl get pods

While the pod is being created, you will see "ContainerCreating".

Finally, to inspect the output of your pod, you can use:
kubectl logs claim-example

In my case, that results in the output:
![image](https://github.com/heatherbaier/dist-ml/assets/7882645/e7def3cd-fda0-4cbf-aa10-79cad37a05c7)

Showing that I have 500 GB of space.

Note that pods with claims will take a few seconds longer to spin up, as K8S must attach the appropriate disks to your pod.  

## Creating an example persistent file
Now that we have a pod running with access to our persistent volume, we can log into that pod in an interactive job, create a file, then delete the pod to illustrate that the content remains available.  To do this, first type in:
```
kubectl exec -it claim-example -- /bin/bash
```

Once you are logged into the pod, we are going to create two files - one in the normal file system (that will be destroyed), and another in the persistent file system (that will be retained).  First let's create the file we know will be destroyed:
```
root@claim-example:~# echo "This file is a goner" > ~/doomedFile
root@claim-example:~# cat ~/doomedFile
This file is a goner
```

Now, let's create a file in our persistent volume:
```
root@claim-example:~# echo "This file will live on" > /kube/home/persistentFile
root@claim-example:~# cat /kube/home/persistentFile 
This file will live on
```

Note the path here - /kube/home .  This is the same path that's specified in our yml file as the mount point for our persistent volume:
```yaml
  containers:
    - name: conda-container
      image: "nvidia/samples:vectoradd-cuda11.2.1"
      volumeMounts:
        - name: home-volume
          mountPath: /kube/home/
```

Now that we have our files, let's destroy the pod and check it worked.  Type `exit` to get out of the pod, then `kubectl delete pod claim-example` to delete the pod.  Once it's deleted, create it again using `kubectl apply -f persistence.yml`, and log back in with `kubectl exec -it claim-example -- /bin/bash`.  If you try to cat the two files we just created, you'll now see:
```
root@claim-example:/# cat ~/doomedFile
cat: /root/doomedFile: No such file or directory
root@claim-example:/# cat /kube/home/persistentFile 
This file will live on
```

# Installing Conda in our Persistent Directory
In order to get conda environments working, we'll want to install them into our persistent volume claim.  To do this, we can use the below yaml code.  Most of this is code you've seen before - but there are a few notable differences.  First, we aren't explicitly requesting resources, as we don't really care what we get here - anything from 1 CPU up will work.  Second, we're both loading our base image (the NVIDIA base), and also installing things within that base image - in this case, the "wget" tool using apt-get.  This is because wget doesn't exist on our base image, and we need it to install conda (note we also have to defined a keyserver here - this is because of some unique issues related to the NVIDIA image, and would not be common to all cases). Finally, and most importantly, we download and install conda to the path we specify with an environmental variable.  If this path is on your persistent volume, then it will stay installed between sessions.

> Of note, this particular yaml does not have a sleep command at the end.  Thus, once conda is installed, it will release the resources for the pod back to the cluster.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: conda-install
spec:
  restartPolicy: OnFailure
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
      command:
        - /bin/bash
        - -c
        - |
          # Add the NVIDIA GPG key
          apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys A4B469963BF863CC
          apt-get update && apt-get install -y wget --fix-missing

          # Define Miniconda installation path
          MINICONDA_PATH="/kube/home/.envs/conda"

          # Check if Miniconda binary exists
          if [ -f "$MINICONDA_PATH/bin/conda" ]; then
              echo "Miniconda binary found. Checking if it is functional."
              PATH="$MINICONDA_PATH/bin:$PATH"
              if conda --version; then
                  echo "Miniconda is functional. Skipping installation."
                  exit 0
              else
                  echo "Miniconda binary is not functional. Reinstalling."
                  rm -rf "$MINICONDA_PATH"
                  wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O /tmp/miniconda.sh
                  bash /tmp/miniconda.sh -b -p "$MINICONDA_PATH"
                  rm /tmp/miniconda.sh
                  PATH="$MINICONDA_PATH/bin:$PATH"
              fi
          else
              echo "Miniconda binary not found. Installing Miniconda."
              wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O /tmp/miniconda.sh
              bash /tmp/miniconda.sh -b -p "$MINICONDA_PATH"
              rm /tmp/miniconda.sh
              PATH="$MINICONDA_PATH/bin:$PATH"
          fi

          #Confirm Installation
          conda info
```

Go ahead and deploy this yml file via `kubectl apply -f 1_installConda.yml`.  Once complete, it will install conda into your persistent claim to the mounted path /kube/home/.envs/conda .  You can monitor the progress of the conda installation by typing either `kubectl get pods` or `kubectl logs conda-install`.

> Because we do not have a sleep command in this script, when it is done the `kubectl get pods` will show that the pod has completed:
![image](https://github.com/heatherbaier/dist-ml/assets/7882645/cba2187c-cb76-491e-93ea-042b9789027d)

# Exploring persistence with Conda
Now that conda is installed in our persistent directory, we can use it in any pod we want.  You can do this interactively, or through a dedicated pod. We'll cover the dedicated pod approach in the next tutorial, but here we'll show how you can do it interactively.

First, create a simple pod that we can log in to that has our persistent volume attached to it, and adds the conda path to our global path:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: interactive-conda
spec:
  activeDeadlineSeconds: 1800  # Pod will be terminated after 30 minutes
  restartPolicy: OnFailure
  volumes:
    - name: home-volume
      persistentVolumeClaim:
        claimName: dsmr-vol-01  # Ensure this is your correct PVC
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
      command:
        - /bin/bash
        - -c
        - |
          sleep infinity
```
Once that file is created, run `kubectl apply -f interactiveCondaPod.yml` to deploy our pod.  Once it's up and running, hop into an interactive shell with `kubectl exec -it interactive-conda -- /bin/bash` .  In order to use conda interactively, we must first tell the shell where it is located.  To do this, simply type these two lines into your shell.  Note this has to be done any time you open a new shell.
```
export MINICONDA_PATH="/kube/home/.envs/conda"
export PATH="$MINICONDA_PATH/bin:$PATH"
```
After you export these paths, you can now type "conda", and you will now see the usual conda help documentation.  Let's go ahead and create a very simple python environment that has pandas installed:
```
conda create -n pandasEx python==3.10
source activate pandasEx
conda install pandas
```
Note that you will frequently need to use the `source activate` technique in kubernetes pods due to system restrictions on rights, but depending on your image `conda activate` may also sometimes work.  Once you have pandas installed, double check it's working by:
```
python
import pandas
```

Finally, to illustrate persistence, log out of the node and delete it using `kubectl delete pod interactive-conda`.  Now, let's recreate it and confirm conda is still working:
```
kubectl apply -f interactiveCondaPod.yml
kubectl exec -it interactive-conda -- /bin/bash
export MINICONDA_PATH="/kube/home/.envs/conda"
export PATH="$MINICONDA_PATH/bin:$PATH"
source activate pandasEx
python
import pandas
```

If everything worked correctly, pandas should import without any additional installs or information required!
