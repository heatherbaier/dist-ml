# PyTorch + DASK

## pyTorch Distributed

This guide walks through how you can quickly distribute deep learning models using pyTorch in a distributed environment. It assumes you have already completed the [basic Python & MPI tutorial](../distributed-ml-with-pytorch/parallel-computing/launching-parallel-programs-on-the-hpc-using-mpi.md), [Python & Dask tutorial](../distributed-ml-with-pytorch/parallel-computing/dask\_intro.md), and understand the [basics of torch](../distributed-ml-with-pytorch/distributed-ml-with-pytorch/torchBasics.md).

## High level "what's going on here"

We're going to be relying heavily on PyTorch's "torch.nn.parallel.DistributedDataParallel", using Dask to request the resources on our nodes. This strategy will effectively send different parts of our dataset to different nodes - i.e., different "batches" - and solve for those in a distributed way. A master will then aggregate the results (gradients, by taking the mean), provide the solution for an Epoch, update the weights, send the new model out to the nodes, and repeat for each Epoch.

This strategy for parallelization is a bit different from what we present in our scikit-learn examples - i.e., here we're going to be fitting a single model using all of our nodes, instead of multiple models with different hyper-parameters.

## Requirements

`conda install -c pytorch pytorch=1.12.1 torchvision=0.13.1`

`conda install dask pandas`

`conda install dask-jobqueue -c conda-forge`

You should have the UC Merced satellite imagery downloaded, as specified in the [basics of torch](../distributed-ml-with-pytorch/distributed-ml-with-pytorch/torchBasics.md) tutorial.

## Job File

As before with our dask job files, there isn't much special here. We're only requesting one node, which will put our other requests in for us.

Because we will be printing to a file to generate our log, you may also want to familiarize yourself with the linux terminal `watch` command.

Note that if you have to kill your job, you can type in: `qstat -u <yourusername>`

Identify the jobID that is asscoiated with the name specified in this job script - here it is demojob. You can then do: `qdel <jobID>` to kill the job. This will then kill all Dask jobs as well (though it takes a minute for them to clear out).

```bash
#!/bin/tcsh
#PBS -N demojob
#PBS -l nodes=1:vortex:ppn=12
#PBS -l walltime=03:00:00
#PBS -j oe

source "/usr/local/anaconda3-2021.05/etc/profile.d/conda.csh"
module load anaconda3/2021.05
module load python/usermodules

unsetenv PYTHONPATH

conda activate torchEnv

cd torchEX
python distTorch.py >& out_dask.out
```

## Directory structure

To run succesfully, the python file will need to be modified in a few ways:

1. You need to point the data loader towards where you unzipped the images.
2. You need to create a "checkpoints" folder, and point the script to where it is. This is where models are saved on disk each epoch.
3. You need to change a few paths for file ouputs, including for a log file and a file that the nodes use to exchange an IP address.

## The Python File

All of the magic is done within Python, courtesy of Dask. The below is heavily commented, so I'll let those comments speak for themselves!

Note that by default it is running 5 epochs using 5 nodes - this may take a little while, so you can consider scaling.

```python
#Torch packages
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import Dataset, DataLoader
from torchvision import datasets
import torch.nn.functional as F
from torchvision import models

#Packages to handle the distribution
from dask.distributed import Client
import torch.distributed as dist
from dask_jobqueue import PBSCluster
import torch.distributed as dist
from torch.nn.parallel import DistributedDataParallel as DDP
from torch.utils.data.distributed import DistributedSampler

#System modules
import os
import time
import socket

PROCESSES = 60
EPOCHS_COUNT = 5

def logger(string, path="/sciclone/home20/dsmillerrunfol/torchEX/log.dan"):
    string = str(string)
    with open(path, "a") as f:
        f.write(string + "\n")

runID = str(int(round(time.time(),0)))

cluster_kwargs = {
    "name": "exDaskCluster",
    "shebang": "#!/bin/tcsh",
    "resource_spec": "nodes=1:c18a:ppn=12",
    "walltime": "01:00:00",
    "cores": 12,
    "processes": 12,
    "memory": "32GB",
    "interface": "ib0",
}


cluster = PBSCluster(**cluster_kwargs)
cluster.scale(PROCESSES)
client = Client(cluster)
client.wait_for_workers(PROCESSES)

rank = [client.scatter(rank) for rank in range(2)]
world_size = [client.scatter(2) for _ in range(2)]
logger(rank)
logger(world_size)


#Network
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

def initDist(rank, size, runID, EPOCHS_COUNT, backend='gloo'):
    from torchvision import transforms

    BATCH_SIZE = 4

    """
    Initialize the distributed environment
    """
    if(rank == 0):
        ip = socket.gethostbyname(socket.gethostname())
        with open("/sciclone/home20/dsmillerrunfol/torchEX/ip_"+ runID +".dan", "w") as f:
            f.write(ip)
    else:
        ip = None
        while ip == None:
            try:
                with open("/sciclone/home20/dsmillerrunfol/torchEX/ip_"+ runID + ".dan", "r") as f:
                    ip = f.readlines()[0]
            except:
                print("Waiting for master node to connect.")
                time.sleep(1)

    def logger(string, path="/sciclone/home20/dsmillerrunfol/torchEX/log.dan"):
        string = str(string)
        with open(path, "a") as f:
            f.write(string + "\n")

    os.environ['MASTER_ADDR'] = ip #if using multiple workers, set the address of the worker with rank = 0
    os.environ['MASTER_PORT'] = "23456"
    os.environ['WORLD_SIZE'] = str(size)
    os.environ['RANK'] = str(rank)

    ## Init distributed
    logger(str(rank) + ": Waiting for all workers to connect...")
    dist.init_process_group(
        init_method="env://",
        backend='gloo'
    )
    device = torch.device('cpu')  
    dist.barrier()

    #At this point, our cluster is initialized - i.e., 
    #the dist.init_process_group will have run on all nodes.
    #Code is blocked from proceeding until that happens.
    #We can now use special torch functions that only work
    #in distributed cases.

    #First, we need to distribute our actual model architecture to all nodes.
    #This is only done once.
    logger(str(rank) + ": Loading model.")
    model = models.resnet101(pretrained=False)
    ddp_model = DDP(model, device_ids=None) #For GPUs, you have to specify device IDs here.
    logger(str(rank) + ": Model loaded.")

    #Now, we need to distribute our data across all the nodes.
    #This is done by creating sub-samples and sending a sub-sample to each node.
    #DistributedSampler handles this process.
    logger(str(rank) + ": Loading imagery.")
    transforms = transforms.Compose([transforms.Resize((64,64)),
                                 transforms.ToTensor()])
    images = datasets.ImageFolder("/sciclone/home20/dsmillerrunfol/torchEX/mercerImages", transform=transforms)
    s = DistributedSampler(images)
    loader = DataLoader(images, batch_size=BATCH_SIZE, sampler=s)
    logger(str(rank) + " Dataset size:" + str(len(loader)))

    #Now we can (finally) train our model like usual!
    lossFN = nn.CrossEntropyLoss()
    optimizer = optim.SGD(ddp_model.parameters(), lr=.01)
    epochs = EPOCHS_COUNT
    num_of_batches = len(loader)
    
    for epoch in range(epochs):
        logger(str(rank) + ": starting epoch " + str(epoch) + " | Batches in Epoch: " + str(num_of_batches))

        #This allows us to shuffle our data
        loader.sampler.set_epoch(epoch)

        ddp_model.train()

        for batch, (X,y) in enumerate(loader):
            X, y = X.to(device), y.to(device)
            #Make our predictions and calculate loss
            forwardPassPred = ddp_model(X)
            loss = lossFN(forwardPassPred, y)

            #Backpropogate
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()

            if batch % 10 == 0:
                loss, current = loss.item(), batch*len(X)
                logger(str(rank) + "||  Loss: " + str(loss) + " | " + str(current) + " of " + str(len(X) * num_of_batches))

        if rank == 0:
            torch.save(ddp_model.module.state_dict(),"/sciclone/home20/dsmillerrunfol/torchEX/checkpoints/" + str(runID) + "_" + str(epoch)+".checkpoint")

        #Prevents any node from continuing to optimize our model before it's done saving as the end of an epoch.
        dist.barrier()

    dist.destroy_process_group()
    #You can return anything you want here - we've saved our weights already.
    #This just goes back to Dask to confirm the process completed.
    #If you added "print(results)" after the client.gather, you see these returns.
    return(str(loss))

future = client.map(initDist, list(range(PROCESSES)), [PROCESSES]*PROCESSES, [runID]*PROCESSES, [EPOCHS_COUNT]*PROCESSES)
logger(future)
results = client.gather(future)

#And, we're done with Dask!  We can now explore our model fit itself, but shut down our
#Dask nodes as they aren't doing anything now:
client.close()

#Now that we have our model, we can load it and run accuracy statistics on it.
#We're just going to run this on our "master" node - i.e., the same node we used
#to spinup the Dask pieces.
from torchvision import transforms
import pandas as pd

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

#Note we're not using the DDP model or loader here,
#as everything is now just running (in forward-pass mode) 
#on the single node.  Also no sampler for our dataset, 
#as we want to test agains the whole thing.
#(Note: in practice we would want a test/train split,
#but I haven't created two datasets for that purpose here.)
model = models.resnet101(pretrained=False)

#Now we need to load the weights for the network we just solved in Dask-world:
model.load_state_dict(torch.load("/sciclone/home20/dsmillerrunfol/torchEX/checkpoints/" + str(runID) + "_" + str(EPOCHS_COUNT-1)+".checkpoint"))

#Load our image data:
transforms = transforms.Compose([transforms.Resize((64,64)),
                                 transforms.ToTensor()])
images = datasets.ImageFolder("/sciclone/home20/dsmillerrunfol/torchEX/mercerImages", transform=transforms)
loader = DataLoader(images, batch_size=4)

#Create our crosstabulation matrix:
logger(accuracyStatistics(model, loader))





print("======SHUTDOWN INITIALIZING=======")
print("==================================")
print("==================================")
print("==================================")
print("==================================")
print("==================================")
print("==================================")
print("==================================")
print("==================================")
print("==================================")
print("==================================")
print("==================================")
print("==================================")
print("======SHUTDOWN INITIALIZING=======")
```
