# Distribution Across Multiple Nodes with sklearn
While single-node processing can be very powerful for solving tasks quickly, sometimes you may need to use distribution across large numbers of nodes.  A simple example of a case where large-scale distribution may be necessary would be a hyperparameter sweep, in which you are attempting to determine optimal hyperparameters for a given training and test dataset.

Here at William & Mary, we use something called [Message Passing Interface](parallel-computing/mpi-and-python.md), which allows nodes to communicate with one another.  This makes it very easy to implement parameter sweeps in a way similar to what we did for [single node parallelization](distributed-scikit/randomSearch.md).  The big advantage to using MPI is that you can scale to an arbitrary number of nodes - i.e., we could use all nodes on a given subcluster of SciClone if we wanted to!

# Pre-requisites
This tutorial assumes you've already implemented the basic MPI approaches described in the section [Launching Programs on the HPC using MPI](parallel-computing/launching-parallel-programs-on-the-hpc-using-mpi.md).

In order to use MPI from a python environment, we need a package called mpi4py.  You can install it with conda by using `conda install -c conda-forge mpi4py`.  

# Job Script
Our job script will be very similar to what we had in the parallelization case, but with a few key differences.
```bash

```

