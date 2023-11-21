# Setting up Torch
Here, we'll install Torch into a conda environment, but we'll do it using a pod dedicated to the install.  Unlike in the previous example, we won't be accessing conda interactively.

## Installing Conda Packages with a Dedicated Pod
In order to install a conda package with a dedicated pod, we'll need to do all of the usual installation steps in an automated fashion.  This can be a fantastic way to setup your environments, as it allows you to quicky construct / deconstruct initial environments and test that they worked in a single script.  Note this script took about 15 minutes to run in testing.

For this example, we'll use the following yml file:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pytorch-install
spec:
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
          # Set the Miniconda path and initialize
          export MINICONDA_PATH="/kube/home/.envs/conda"
          export PATH="$MINICONDA_PATH/bin:$PATH"

          # Check if the conda environment already exists
          if conda info --envs | grep -q "torchEnv"; then
              echo "Conda environment 'torchEnv' already exists. Deleting and rebuilding."
              conda env remove -n torchEnv --yes
          fi

          # Create a new conda environment with Python 3.11
          conda create -y -n torchEnv python=3.11

          # Activate the environment
          source activate torchEnv

          # Install PyTorch, torchvision, and pytorch-cuda
          conda install -y -n torchEnv pytorch torchvision pytorch-cuda=12.1 -c nvidia -c pytorch


          #Test that we have a GPU, and it's registering in Torch.
          echo "Testing visibility of GPUs on system and in Python."
          python -c "import torch; gpus = torch.cuda.device_count(); print(f'Available GPUs: {gpus}'); [print(f'GPU {gpu}: {torch.cuda.get_device_name(gpu)}') for gpu in range(gpus)]"
          nvidia-smi

          # Check for GPU availability using Python and PyTorch
          GPU_COUNT=$(python -c "import torch; print(torch.cuda.device_count())")

          # Export the Conda environment if at least one GPU is detected
          if [ "$GPU_COUNT" -gt 0 ]; then
              echo "At least one GPU detected, exporting Conda environment."
              echo "Exporting environment yaml file to:" $MINICONDA_PATH
              conda env export -n torchEnv > $MINICONDA_PATH/torchEnv.yml

          else
              echo "No GPUs detected.  Something may have gone wrong, or you may not have asked for any in your pod."
          fi

```
Notably:
- We are requesting a GPU.  This is so we can test that python can actually detect and load the GPUs for processing.
- In our script, we are installing the appropriate version of pytorch-cuda (12.1) for the NVIDIA drivers available in our base image.
- Finally, we have some short diagnostic code that loads torch within python and confirms it can detect GPUs.
- This script has no sleep, so will terminate after completion.

Go ahead and create this yml file, and apply it with `kubectl apply -f 2_createTorchEnv.yml`. As before, you can watch the progress of the pod with `kubectl get pods` and `kubectl logs pytorch-install`.  If it is taking a while, you can also use `kubectl describe pod pytorch-install` to get a bit more information on the status of the pod.  Note that this pod make take a while before it registers as complete, as the relevant packages are larger than a gig in total for install size.  When the status of the pod changes from "Running" to "Complete", you'll know its done; alternatively, in the logs, you should see information about the total number of GPUs accessible by python at the end of the script after it completes.
> Sometimes, you may want to run `watch kubectl get pods` on your frontend to monitor pods that may take a while to complete.

If everything works correctly, after about 15 minutes `kubectl logs pytorch-install` should return something similar to:
![image](https://github.com/heatherbaier/dist-ml/assets/7882645/67284660-7291-4a3f-a0fa-47f8a4f28a4b)

