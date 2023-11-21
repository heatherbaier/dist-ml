# Running Torch with Real Data in a Pod
Now that we have torch installed in a persistent conda environment, we want to illustrate how to copy files from a local machine (i.e., the frontend) directly into our persistent volume.  You can use this to transfer data from anywhere you can mount on your local frontend up into the K8S environment.  We'll start with a simple python script.

First, we need to deploy a pod that has access to the persistent volume we want to upload something to.  A simple example would be:
```
apiVersion: v1
kind: Pod
metadata:
  name: file-passthrough
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
This example pod has access to our persistent volume, and doesn't really do much else.  Go ahead and deploy it with `kubectl apply -f persistentVolumeUpload.yml`.

While that container is created, go ahead and create a python file named `gpu.py` by copying the below.  Create it on the frontend - i.e., cm, not within the pod. This file downloads an image dataset (CIFAR10) into the directory /kube/data, and then runs a few epochs to classify it using a very small convolutinal net:

```python
import torch
import torch.nn as nn
import torch.nn.functional as F 
import torch.optim as optim
import torchvision
import torchvision.transforms as transforms
import time

devs = [torch.device("cuda:0" if torch.cuda.is_available() else "cpu"), "cpu"]

for device in devs:
    # Define the CNN model
    class Net(nn.Module):
        def __init__(self):
            super(Net, self).__init__()
            self.conv1 = nn.Conv2d(3, 6, 5)
            self.pool = nn.MaxPool2d(2, 2)
            self.conv2 = nn.Conv2d(6, 16, 5)
            self.fc1 = nn.Linear(16 * 5 * 5, 120)
            self.fc2 = nn.Linear(120, 84)
            self.fc3 = nn.Linear(84, 10)

        def forward(self, x):
            x = self.pool(F.relu(self.conv1(x)))
            x = self.pool(F.relu(self.conv2(x)))
            x = x.view(-1, 16 * 5 * 5)
            x = F.relu(self.fc1(x))
            x = F.relu(self.fc2(x))
            x = self.fc3(x)
            return x

    net = Net().to(device)

    # Define the loss function and optimizer
    criterion = nn.CrossEntropyLoss()
    optimizer = optim.SGD(net.parameters(), lr=0.001, momentum=0.9)

    # Load CIFAR-10 dataset
    transform = transforms.Compose([transforms.ToTensor(), transforms.Normalize((0.5, 0.5, 0.5), (0.5, 0.5, 0.5))])
    trainset = torchvision.datasets.CIFAR10(root='/kube/data', train=True, download=True, transform=transform)
    trainloader = torch.utils.data.DataLoader(trainset, batch_size=4, shuffle=True, num_workers=2)

    # Train the model
    start_time = time.time()

    for epoch in range(5):  # Loop over the dataset multiple times
        epoch_start_time = time.time()
        running_loss = 0.0
        for i, data in enumerate(trainloader, 0):
            inputs, labels = data[0].to(device), data[1].to(device)
            optimizer.zero_grad()  # Zero the parameter gradients
            outputs = net(inputs)  # Forward
            loss = criterion(outputs, labels)
            loss.backward()  # Backward
            optimizer.step()  # Optimize
            running_loss += loss.item()
            if i % 2000 == 1999:  # Print every 2000 mini-batches
                print('[%d, %5d] loss: %.3f' % (epoch + 1, i + 1, running_loss / 2000))
                running_loss = 0.0
        epoch_end_time = time.time()
        print("Epoch %d completed in %s seconds" % (epoch+1, round(epoch_end_time - epoch_start_time, 2)))

    end_time = time.time()
    total_time = end_time - start_time
    print("Trained on: " + str(device))
    print("Total training time: %s seconds" % round(total_time, 2))
    print("----------------------------------")
    print("----------------------------------")
```

Once you've created the python file, confirm that your new file-passthrough pod is up and running with `kubectl get pods`.  If it's running, you can now copy the GPU script into your persistent volume.  To do so, you can run:
```
kubectl cp gpu.py dsmillerrunfol/file-passthrough:/kube/home/gpu.py
```

Once you run that command, we can run a command against the file-passthrough pod to make sure it copied correctly:
```
kubectl exec -it file-passthrough -- ls -la /kube/home
```
If succesful, you should see something like this, including the new gpu.py file:
![image](https://github.com/heatherbaier/dist-ml/assets/7882645/f123255f-5b44-458e-92e8-109b2f71e3d2)

# Running the python file on GPUs
Once your file is copied over, you can shutdown the file-passthrough pod and launch the GPU pod that will actually process the data.  For now, we'll only request a single GPU in our pod, and 2 CPUs to help with batching:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: torch-test
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
          cpu: "2"
        limits:
          memory: "32Gi"
          nvidia.com/gpu: 1
          cpu: "2"
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
          python /kube/home/gpu.py
```

Go ahead and submit that with `kubectl apply -f 3_torchWithData.yml`, and then monitor the progress of torch using `kubectl logs torch-test`.  Note it may take around a minute for logs to start.  If the script works correctly, you should see CIFAR downloading, followed by a classification across a few epochs of training:
![image](https://github.com/heatherbaier/dist-ml/assets/7882645/3e40f672-28da-41a3-baa2-a32895d3a120)

