# pyTorch Distributed

This guide walks through how you can quickly distribute deep learning models using pyTorch in a distributed environment.  It assumes you have already completed the [basic Python & MPI tutorial](parallel-computing/launching-parallel-programs-on-the-hpc-using-mpi.md), and understand the [basics of torch](distributed-ml-with-pytorch/torchBasics.md).

# High level "what's going on here"

We're going to be relying heavily on PyTorch's "torch.nn.parallel.DistributedDataParallel".  This strategy will effectively send different parts of our dataset to different nodes - i.e., different "batches" - and solve for those in a distributed way.  A master will then aggregate the results (gradients), provide the solution for an Epoch, update the weights, send the new model out to the nodes, and repeat for each Epoch.

This strategy for parallelization is a bit different from what we present in our scikit-learn examples - i.e., here we're going to be fitting a single model using all of our nodes, instead of multiple models with different hyper-parameters.  

# Requirements
`conda install -c pytorch pytorch=1.12.1 torchvision=0.13.1`

You should have the UC Merced satellite imagery downloaded, as specified in the [basics of torch](distributed-ml-with-pytorch/torchBasics.md) tutorial.

# Job File
