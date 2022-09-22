# What is Dask?

Dask is an alternative package to MPI and Torque for distributed computing.  We can leverage it locally on our clusters using Dask on top of Torque - i.e., Dask requests resources from Torque dynamically.  In the future, we hope to be able to run Dask natively (i.e., without Torque at all).  Today, using Dask requires a dual-layer job queue, in which first you use Torque to submit a master Dask job, and then second the Dask master uses Torque to schedule resources.  The advantage is that Dask plays very nicely with many widely used packages (i.e., Pandas, sklearn), and has many of it's own highly distributed implementations of ML algorithms.  This can greatly reduce the complexity of your implementations, or allow you to more flexibly take advantage of all resources for a given job.

# Installing Dask

To use dask in the HPC environment, you need to install both the jobqueue module (which handles scheduling with Torque) and dask itself (which handles intra-node communication).  Dask is unique as compared to MPI in that it handles scheduling itself - i.e., you will not request the nodes or processors you want upfront, but rather launch a python file that then requests resources through dask.

`conda install dask`

`conda install dask-jobqueue -c conda-forge`

This example was tested with dask 2022.9.1 and dask-jobqueue 0.8.0.

# Master job script

The job script you directly submit will run one job, on a master node.  This job will execute a python script, which will then itself launch other jobs.

The master job script is similar to any other script you may have written:
```sh
#!/bin/tcsh
#PBS -N demojob
#PBS -l nodes=1:vortex:ppn=12
#PBS -l walltime=00:30:00
#PBS -j oe

source "/usr/local/anaconda3-2021.05/etc/profile.d/conda.csh"
module load anaconda3/2021.05
module load python/usermodules

unsetenv PYTHONPATH

conda activate aml35

cd /sciclone/home20/dsmillerrunfol/myPythonFileDirectory
python dask_example.py >& out_dask.out
```

# Python file

The python file is where additional jobs will be launched to solve for whatever task you are interested in.  Here we're going to do a very simple example of chaining two functions together across an arbitrary number of processes.

```python
from dask.distributed import Client
from dask_jobqueue import PBSCluster

#These are the arguments you would normally specify in your job file.
#Effectively, dask will be running a qsub *for* you, numerous times, as 
#per your specifications.
#Of note, cores and processes should be similar to what you request
#in your resource spec.  Here, I have one process per core (no hyperthreading),
#but in practice most of the cores on the HPC can have 2 processes each, 
#so setting processes to 24 would be a more effective allocation if you have
#enough memory.  Note we're requesting to use all the memory on each Vortex node (32gb).
cluster_kwargs = {
    "name": "exDaskCluster",
    "shebang": "#!/bin/tcsh",
    "resource_spec": "nodes=1:c18a:ppn=12",
    "walltime": "00:20:00",
    "cores": 12,
    "processes": 12,
    "memory": "32GB",
    "interface": "ib0",
}

cluster = PBSCluster(**cluster_kwargs)
#Here we tell the program how far to scale our cluster, and this is where the magic comes in.
#If we have 12 core / 12 process machines, and ask to scale to 48, this single line of code
#will automatically launch 4 jobs on the HPC for 4 more nodes (4 * 12 = 48).  These are our
#worked nodes.  Also note your walltimer starts ticking as soon as you run this line.
cluster.scale(48)
client = Client(cluster)

#Dask works by running individual functions on each node, and then aggregating the results.
#So, any code we want to write needs to be defined as a function, i.e.:
def multiply(x):
    return(x * 2)

def neg(x):
    return(-x)

#Now we need to pass some input into each function.  Here we create a list of 1's, with 48 entries, i.e.:
#[1, 1, 1, 1, 1, ....., 1, 1, 1, 1, 1]
#Dask will pass one of those "1"s to each of our 48 processes through the client.map command.
#It will then solve for the results for A and B.
#Importantly, it will not actually *run* this until you run line 91 - i.e., you need to submit
#the chain of functions you want to solve for.  This will allow each individual process to solve A, then B, without
#transmitting any results back to the master, which is far more effecient.
#This means that when you print(A), you get a Future - i.e., something that is pending until a submission.
A = client.map(multiply, [1]*48)
print(A)
B = client.map(neg, A)
total = client.submit(sum, B)
print("OUTPUT:")
print(total.result())
#Note that because of how our torque system interacts with dask, you will always get a "cancelled" error
#(sometimes "futures" or both) at the end of a dask script.  These lines are just so you can see your outputs
#above the error.
print("======================")
print("======================")
print("END OF OUTPUT")
print("======================")
print("======================")
```
