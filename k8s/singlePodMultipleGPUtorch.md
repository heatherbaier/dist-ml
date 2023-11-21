# Single Pod, Multi-GPU with Torch
Here, we'll be requested 3 GPUs and running a torch classification on CIFAR using them.  Note that this requires all 3 GPUs are physically on the same node, as our resource request for a pod will specify 3 GPUs, and pod resources can't extend beyond your physical computational capacity.  We'll show how to create multiple 1-GPU pods that can exist on any node, and how to fit a model using that distribution strategy, in the next tutorial.

# YAML Resource Request
Here, we're going to simply increase the number of GPUs that we request to 3, and allocate more cores to help load data onto the GPUs.  Additionally, we're going to change the script we use for processing to `multiGPU.py`, which will be detailed below.  

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

Next, we're going to make some small modifications to our python classification code to enable the model to be parallelized across multiple GPUs.  Create this file, save it as multiGPU.py, and then copy it to your persistent volume using `kubcp multiGPU.py dsmillerrunfol/file-passthrough:/kube/home/multiGPU.py`.  You may need to restart your file-passthrough pod to do this.  Your new python file will look like:
```python
import torch
import torch.nn as nn
import torch.optim as optim
import torchvision
import torchvision.transforms as transforms
import torch.nn.functional as F
import time

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

# Check if multiple GPUs are available and wrap the model using DataParallel
print("Creating and adding data parallel to model.")
net = Net()
if torch.cuda.device_count() > 1:
    print("Using", torch.cuda.device_count(), "GPUs!")
    net = nn.DataParallel(net)

print("Sending to CUDA")
net.to(torch.device("cuda" if torch.cuda.is_available() else "cpu"))

# Define the loss function and optimizer
criterion = nn.CrossEntropyLoss()
optimizer = optim.SGD(net.parameters(), lr=0.001, momentum=0.9)

print("Loading CIFAR10")
# Load CIFAR-10 dataset
transform = transforms.Compose([transforms.ToTensor(), transforms.Normalize((0.5, 0.5, 0.5), (0.5, 0.5, 0.5))])
trainset = torchvision.datasets.CIFAR10(root='/kube/data', train=True, download=True, transform=transform)

print("Initializing data loader")
trainloader = torch.utils.data.DataLoader(trainset, batch_size=32, shuffle=True, num_workers=2)

# Train the model
start_time = time.time()
print("Starting training.")
for epoch in range(5):  # Loop over the dataset multiple times
    epoch_start_time = time.time()
    running_loss = 0.0
    for i, data in enumerate(trainloader, 0):
        inputs, labels = data[0].to(torch.device("cuda" if torch.cuda.is_available() else "cpu")), data[1].to(torch.device("cuda" if torch.cuda.is_available() else "cpu"))
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
print("Total training time: %s seconds" % round(total_time, 2))
print("----------------------------------")
print("----------------------------------")

```
With the core new element being:
```
if torch.cuda.device_count() > 1:
    print("Using", torch.cuda.device_count(), "GPUs!")
    net = nn.DataParallel(net)
```
This code replicates the model across all GPUs, and then subsets each batch into further mini-batches that get distributed across the GPUs (so, in this example, each of the 3 GPUs processes 1/3 of the batch size of 36, for a per-card batch of 12).  Each GPU solves for it's gradients, and then the master synchs these gradients to update the model, and sends the new weights out to each GPU each batch. 

Once your multiGPU.py script has been uploaded, you can `kubectl apply -f 4_multiGPUTorch.yml`.  
