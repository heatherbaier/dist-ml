# What is Torch?
Torch is a very popular library for handling matrix operations on GPU (similar to numpy, but with a focus on GPU support).  The most common way to use Torch - at least in this authors world - is through a wrapper called PyTorch.  This brief guide will show an example of implementing code in PyTorch on our cluster, but we won't be distributing it just yet.

# Example Data
Our example dataset is going to be a satellite dataset from UC Merced.  You can download it from:
https://geolab.wm.edu/assets/static_files/mercerImages.zip

Here is some quick python starter code you can use to grab and extract the data:

```python
import requests
import shutil
r = requests.get("https://geolab.wm.edu/assets/static_files/mercerImages.zip")

open("mercerImages.zip", 'wb').write(r.content)
shutil.unpack_archive("mercerImages.zip")
```

Note, for this example I have created a new folder "torchEX" in my home directory.  You do not have to do this, but my examples will presume you did.  If you do not, you'll need to edit your filepaths accordingly.  I also extract the folder as per the lines above - i.e., you'll have a folder called "mercerImages", with ~20 folders inside it named "agriculture", "airplane", etc. - all with images inside them.

# Environment
I recommend creating a new environment for using torch, as there are packages and versions that frequently do not play well between sklearn and torch.
conda create -n torchEnv python=3.9
conda activate torchEnv
conda install -c pytorch pytorch=1.12.1 torchvision=0.13.1 pandas

# Basic Torch Net
Here we're going to implement a basic Torch net on a single node in CPU mode.  The bash script is going to be a very simple one - note we're only going to use a single core (-c 1).

```bash j
#!/bin/tcsh
#PBS -N torchDemoJob
#PBS -l nodes=1:vortex:ppn=12
#PBS -l walltime=00:30:00
#PBS -j oe

source "/usr/local/anaconda3-2021.05/etc/profile.d/conda.csh"
module load anaconda3/2021.05
module load python/usermodules
module load mvapich2-ib

unsetenv PYTHONPATH

conda activate torchEnv

cd torchEX
mvp2run -c 1 python3 basicTorch.py >& torch_output.out
```

The python is where all of our magic is going to happen (I named my file "basicTorch.py").
Note this network is going to be *terrible*, and is mearly for demonstration purposes.

```python
import torch
from torchvision import datasets
from torchvision import transforms
from torch import nn
import torch.nn.functional as F
from torch.utils.data import DataLoader
import pandas as pd

#We are going to be deploying on CPU-based nodes in our distribution
#tutorial, but if you have GPU's available the switch is fairly
#straightforward:
#device = "cuda" if torch.cuda.is_available() else "cpu"
device = "cpu" 

#We need to load our data in a special way for pyTorch.
#First, we need to define a few transforms - these
#are pre-processing steps that will be applied to all images.
#It is very common to resize images to similar dimensions, for example.
#The last transform (.ToTensor()) loads our data into a format
#that is generally used by GPUs.
transforms = transforms.Compose([transforms.Resize((64,64)),
                                 transforms.ToTensor()])

#Second we tell it where the dataset is using torchvision's handy
#datasets loader. Note you'll have to change this path to wherever
#your image data is living.
images = datasets.ImageFolder("/sciclone/home20/dsmillerrunfol/torchEX/mercerImages", transform=transforms)
#Gives us the statistics on how many images:
print(images)

#Prints out the classes that were detected (and the Ids assigned)
print(images.class_to_idx)

#We're going to implement a very simple
#sequential linear model, in which we'll be inputting
#each image  into weighted linear layers (no convolutions),
#and interspersing ReLu.
#In our data loader above, we rescaled our images to all be 64x64
#(with 3 bands - hence the 3!)
#This is the input into the first linear layer.
#Note the final line (nn.Linear(32,21) - the 21 is the number
#of classes in the UC Merced satellite imagery dataset).
class NeuralNetwork(nn.Module):
    def __init__(self):
        super().__init__()
        self.flatten = nn.Flatten()
        self.simpleNet = nn.Sequential(
            nn.Linear(64*64*3, 512),
            nn.ReLU(),
            nn.Linear(512, 128),
            nn.ReLU(),
            nn.Linear(128, 64),
            nn.ReLU(),
            nn.Linear(64, 32),
            nn.Linear(32, 21)
        )

    def forward(self, x):
        x = self.flatten(x)
        x = self.simpleNet(x)
        return x

model = NeuralNetwork().to(device)
print(model)

#Now we're going to load our data using a dataloader.
#This let's us take only a small number of images at a time
#(i.e., a batch of 8 images) to avoid running out of memory!
loader = DataLoader(images, batch_size=8)

#Now we define our loss function and optimizer
loss_fn = nn.CrossEntropyLoss()
optimizer = torch.optim.SGD(model.parameters(), lr=1e-3)

#And we can implement our training function - this is where we update the weights.
#We'll call this in a loop shortly.
def trainModel(dataLoader, model, loss_fn, optimizer):
    size = len(dataLoader.dataset)
    model.train()
    for batch, (X,y) in enumerate(dataLoader):
        X, y = X.to(device), y.to(device)
        #Make our predictions and calculate loss
        forwardPassPred = model(X)
        loss = loss_fn(forwardPassPred, y)

        #Backpropogate
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()

        if batch % 10 == 0:
            loss, current = loss.item(), batch*len(X)
            print("Loss: " + str(loss) + " | " + str(current) + " of " + str(size))
    
epochs = 20
for t in range(epochs):
    print("Epoch " + str(t))
    trainModel(loader, model, loss_fn, optimizer)

#At this point, we have a model that's been fit across 20 epochs.
#Let's output some stats about our model.

def accuracyStatistics(model, dataLoader):
    model.eval()
    test_loss = 0
    device = 'cpu'

    y_pred = []
    y_actu = []
    with torch.no_grad():
        for data, target in dataLoader:
            data, target = data.to(device), target.to(device)
            output = model(data)
            test_loss += F.nll_loss(output, target, reduction='sum').item()  # sum up batch loss
            pred = output.argmax(dim=1, keepdim=True)  # get the index of the max score
            y_pred.extend(torch.flatten(pred).tolist()) 
            y_actu.extend(target.tolist())
           
    y_pred = pd.Series(y_pred, name='Predicted')
    y_actu = pd.Series(y_actu, name='Actual')
    cm = pd.crosstab(y_actu, y_pred)

    return(cm)

print(accuracyStatistics(model, loader))

```
