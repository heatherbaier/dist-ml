# Distribution Across Multiple Nodes with sklearn
While single-node processing can be very powerful for solving tasks quickly, sometimes you may need to use distribution across large numbers of nodes.  A simple example of a case where large-scale distribution may be necessary would be a hyperparameter sweep, in which you are attempting to determine optimal hyperparameters for a given training and test dataset.

Here at William & Mary, we use something called [Message Passing Interface](parallel-computing/mpi-and-python.md), which allows nodes to communicate with one another.  This makes it very easy to implement parameter sweeps in a way similar to what we did for [our simple implementation](distributed-scikit/randomSearch.md), but with far more granular control over how jobs are distributed to our worker nodes.  The big advantage to using MPI is that you can scale to an arbitrary number of nodes - i.e., we could use all nodes on a given subcluster of SciClone if we wanted to - and be explicit in how we map jobs to each node.

# Pre-requisites
This tutorial assumes you've already implemented the basic MPI approaches described in the section [Launching Programs on the HPC using MPI](parallel-computing/launching-parallel-programs-on-the-hpc-using-mpi.md).

In order to use MPI from a python environment, we need a package called mpi4py.  You can install it with conda by using `conda install -c conda-forge mpi4py`.  

# Job Script
Our job script will be very similar to what we had in the parallelization case, but with a few key differences.  First, note that we're requesting 2 nodes with 12 cores each - so, a total of 24 processors.  Second, we're calling mvapich2-ib as a module, which allows us to start our python file with mvp2run - a convenient wrapper around MPI.

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

# Python Script

The python script has a number of important features, but at it's heart we are trying to do the exact same thing as we did in the simple node parallelization case: create a set of CSVs containing models tested with different parameters (see [the simple random search](distributed-scikit/randomSearch.md)). A few key things to pay attention to:
1) First, we are generating a list of 1000 parameters "C", but creating that list on our master process - i.e., rank 0.
2) Then, we're sending *pieces* of that list out to all of the workers.  This is done by first creating chunks (i.e., slices of the list of parameters), and then sending those chunks out to each worker process.
3) Finally, on each worker process, we're receiving the paramters it needs to test, loading the data, fitting the model, and saving the results into a CSV.

Just like before, we can concatenate the output CSVs using `cat *csv > all.csv`.  If desired, you could also build that part into python by sending the results of the tests back to the rank 0 node, and outputting a single CSV.

```python example.py
from mpi4py import MPI
import pandas as pd
import random
import math

comm = MPI.COMM_WORLD
comm_size = comm.Get_size()
comm_rank = comm.Get_rank()

if(MPI.COMM_WORLD.Get_rank() == 0):
    print("I am the rank 0 node.")
    
    #I can send anything I want to other processes - for example:
    param_list = []
    for i in range(0,1000):
        C = random.random() * 10
        param_list.append(C)
    
    #divide our list by the number of cores we have available.
    #We'll round down
    tasks_per_core = math.ceil(len(param_list) / comm.Get_size())
    chunks = [param_list[x:x+tasks_per_core] for x in range(0, len(param_list), 42)]

    for coreID in range(1,comm.Get_size()):
        comm.send(chunks[coreID-1], dest=coreID, tag=11)


if(MPI.COMM_WORLD.Get_rank() != 0):
    from sklearn.preprocessing import StandardScaler
    from sklearn.linear_model import LogisticRegression
    from sklearn.metrics import accuracy_score
    from time import time

    data = pd.read_csv('studentpor_bigger.csv')

    X = data[["traveltime", "studytime"]].values
    y = data["Walc"]

    scale_X = StandardScaler()
    X = scale_X.fit_transform(X)
    
    parameters = comm.recv(source=0, tag=11)
    print("I (" + str(MPI.COMM_WORLD.Get_rank()) + ") have received " + str(len(parameters)) + " parameters to test.")

    for p in parameters:
        logC = LogisticRegression(penalty="elasticnet", solver="saga", fit_intercept=False, tol=1.0, C=p)
        logC.fit(X, y)

        #Percent accuracy:
        acc = accuracy_score(y, logC.predict(X))
        f = open("/sciclone/home20/dsmillerrunfol/dml/results/" + str(p)+ ".csv", "w")
        f.write(str(p) + "," + str(acc) + "\n")
        f.close()

#Once we're done, we should have a folder full of csv
#files.  All we need to do is concatenate them together
#into one output: cat *csv > all.csv
```