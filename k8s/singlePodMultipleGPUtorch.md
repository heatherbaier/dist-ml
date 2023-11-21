# Single Pod, Multi-GPU with Torch
Here, we'll be requested 3 GPUs and running a torch classification on CIFAR using them.  Note that this requires all 3 GPUs are physically on the same node, as our resource request for a pod will specify 3 GPUs, and pod resources can't extend beyond your physical computational capacity.  We'll show how to create multiple 1-GPU pods that can exist on any node, and how to fit a model using that distribution strategy, in the next tutorial.

# YAML Resource Request
Here, we're going to simply increase the number of GPUs that we request to 3.  Additionally, we're going to change the script we use for processing to `multiGPU.py`, which will be detailed below.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multigpu-torch
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
          nvidia.com/gpu: 3
          cpu: "8"
        limits:
          memory: "32Gi"
          nvidia.com/gpu: 3
          cpu: "8"
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

          # Activate the environment
          source activate torchEnv

          #Make sure our GPUs are loading
          python -c "import torch; gpus = torch.cuda.device_count(); print(f'Available GPUs: {gpus}'); [print(f'GPU {gpu}: {torch.cuda.get_device_name(gpu)}') for gpu in range(gpus)]"

          #Run our gpu torch script
          python /kube/home/multiGPU.py
```

Next, we're going to make some small modifications to our python classification code to enable the model to be parallelized across multiple GPUs.  Create this file, save it as multiGPU.py, and then copy it to your persistent volume using `kubcp multiGPU.py dsmillerrunfol/file-passthrough:/kube/home/multiGPU.py`.  You may need to restart your file-passthrough pod to do this.

Once your multiGPU.py script has been uploaded, you can `kubectl apply -f 4_multiGPUTorch.yml`.  
