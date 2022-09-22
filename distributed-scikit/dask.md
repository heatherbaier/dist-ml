# Dask
This presumes you have already done the basic [dask guide](parallel-computing/dask_intro.md).

Dask is an alternative scheduling and job distribution system that can be flexibly deployed across a wide range of HPC environments.  Writing in dask can allow your script to quickly translate between different HPC environments, at the cost of some increased overhead in some scenarios.

One advantage of Dask is that it has inbuilt capabilities that allow it to play very nicely with sklearn.  This tutorial shows one such example, using a random search across a set of parameters using nearly entirely inbuilt tools.

# The Job Script
The job script only launches a single node - a master node.  This will then spawn other processes.  You do *not* specify the total resources you want here - i.e., you only want this to ever have one node.
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

# The Python Script
Much of our logic for submitting jobs now lives in our python script.  Below, you will see a heavily commented version of a python script which searches across a wide range of hyperparameters to identify the best combination using a dask backend.

```python
from dask.distributed import Client
from dask_jobqueue import PBSCluster
import joblib

import numpy as np
import pandas as pd

from sklearn.model_selection import RandomizedSearchCV
from sklearn.preprocessing import StandardScaler
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import accuracy_score


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
cluster.scale(48)
client = Client(cluster)

#Note that we're doing the below data processing
#on our master node, and then distributing
#the data to each worker.
#This may or may not be desireable - we could instead
#have a function we map to each node that loads our data.
data = pd.read_csv('studentpor.csv')

X = data[["traveltime", "studytime"]].values
y = data["Walc"]

scale_X = StandardScaler()
X = scale_X.fit_transform(X)

#Note the below will create a large search space 
#of 300 possible combinations - 100 C values,
#and 3 loss penalties.
#The RandomizedSearchCV will randomly subset from this n_iter times,
#using a cross-fold validation on that set of parameters.
#You can use GridSearchCV to iterate over every combination if you want as well.
parameters_to_search = {
    'C' : np.arange(0,10,0.1),
    'penalty' : ['elasticnet', 'l1', 'l2'],
}

logC = LogisticRegression(solver="saga")

#n_iter is set to 48 so we spin up one process per process in our cluster.
#You can set this to more - i.e., 96 would result in 2 models being fit on each core 
#(as we have one core == 1 process set now).
#This will result in 3 * 48 total models - 3 folds for 48 parameter sets.
search = RandomizedSearchCV(logC, parameters_to_search, cv=3, n_iter=48, verbose=10)

with joblib.parallel_backend('dask'):
    search.fit(X, y)

print("\n Best estimator:\n", search.best_estimator_)
print("\n Best Score:\n", search.best_score_)
print("\n Best Parameters:\n", search.best_params_)

print("======================")
print("======================")
print("END OF OUTPUT")
print("======================")
print("======================")
```