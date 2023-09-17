# Python & MPI

## Launching Programs on the HPC using MPI

Here at William & Mary, we use something called Message Passing Interface (i.e., MPI), which allows nodes to communicate with one another. This is a relatively simple interface

## Packages

In order to use MPI from a python environment, we need a package called mpi4py. You can install it with conda by using `conda install -c conda-forge mpi4py`.

## Job Script

Our initial job script is very similar to other job scripts you will have seen in this tutorial. A few things to note include:

1. We are asking for 2 nodes and 12 processors per node (ppn). That means we should have 24 cores allocated.
2. We are loading the mvapich2-ib library, which allows us to use mvp2run. mvp2run is a wrapper around MPI, which handles the inter-node communications.

Note you will probably have to change the two "cd" lines to match whatever directory you dropped your scripts into.

```bash
#!/bin/tcsh
#PBS -N demojob
#PBS -l nodes=2:vortex:ppn=12
#PBS -l walltime=00:30:00
#PBS -j oe

source "/usr/local/anaconda3-2021.05/etc/profile.d/conda.csh"
module load anaconda3/2021.05
module load python/usermodules
module load mvapich2-ib

unsetenv PYTHONPATH

conda activate aml35

cd dml
cd skl_dist
mvp2run python example.py >& output.out
```

## Python Script

Here we have an example python script that shows some of the basics of how MPI works. A few things to note:

1. Anything inside the "if" statements will run only on certain processors - i.e., "rank 0" is the first processor.
2. In this example, we're only tasking two of our total 24 processors - i.e., every processors will run the "print" statement at the top, then processor 0 will declare it is rank 0 and create a parameter list. It will then send it to rank 1 (dest=1), with a tag of 11 (you can tag whatever you want).
3. Then, on rank 1 processor, it will load a pandas dataset, and then _receive_ the list of parameters from rank 0 (specifically asking for tag 11).

You can arbitrarily extend this communication logic to implement a wide range of distribution strategies - for example, a [Random Search](distributed-scikit/mpi.md).

```python
from mpi4py import MPI
import pandas as pd
import random

comm = MPI.COMM_WORLD
comm_size = comm.Get_size()
comm_rank = comm.Get_rank()

#Note that comm.Get_size() is going to be based on the number of cores you have in your job script.
#Here it should be 24 (2 nodes, 12 cores each)
print("Hello from " + str(comm.Get_rank()) +".  There are a total of " + str(comm.Get_size()) + " of us.  Good luck.")

if(MPI.COMM_WORLD.Get_rank() == 0):
    print("I am the rank 0 node.  Code you write here will only execute on this process.")
    print("This is frequently used to create a master node that collects results from other nodes.")
    
    #I can send anything I want to other processes - for example:
    param_list = []
    for i in range(0,1000):
        C = random.random() * 10
        param_list.append(C)
    comm.send(param_list[0:100], dest=1, tag=11)

if(MPI.COMM_WORLD.Get_rank() == 1):
    #I want to load the data on individual nodes - 
    #not send it over the network, as that is much slower.
    data = pd.read_csv('studentpor_bigger.csv')
    parameters = comm.recv(source=0, tag=11)
    print(parameters)

```
