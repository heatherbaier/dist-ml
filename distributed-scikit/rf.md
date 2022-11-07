# Random Forests

This presumes that you have already completed the examples in the Random Search section of the guide.

## Introduction & Data
A Random Forest Classifier is made up of many independent classification (or regression) trees, making it a perfect candidate for parallelization across multiple processes.  Following that, scikit-learn has inbuilt options that allow you to fit these trees across multiple processors, if they are available.

We will write a script which does this twice, first running on a single core, and then running across 12 cores.  We will time each one, and return to the user the total runtimes.

To facilitate this comparison, we have created a bigger version of the student alcohol dataset, copying the rows to generate a larger number of observations.  This is helpful, as it slows the algorithms down and allows you to more easily see the difference between single and multi-core implementations.

[studentpor_bigger.csv](https://github.com/heatherbaier/dist-ml/files/9528910/studentpor_bigger.csv)

## Job File
First, we'll need to setup our job file.  Most things here are similar to what you saw in the Random Search example, but with some key differences.

```Job File
#!/bin/tcsh
#PBS -N demojob
#PBS -l nodes=1:vortex:ppn=12
#PBS -l walltime=00:30:00
#PBS -j oe

source "/usr/local/anaconda3-2021.05/etc/profile.d/conda.csh"
module load anaconda3/2021.05
module load python/usermodules
module load mvapich2-ib

unsetenv PYTHONPATH

conda activate aml35

cd dml
cd skl_par
mvp2run -c 1 python simplePar.py >& output.out
```

In this job file, you'll want to note two things.  First, we're only requesting one node.  This is because the inbuilt capability of sklearn to distribute only extends to a single node - i.e., you can spin up processes on a single computer at a time.  We'll go into more advanced techniques for multi-node distribution later.

Also important is in the mvp2run, the command -c 1 .  What this is telling the scheduler to do is to create only *one* version of simplePar.py on the node.  Then, the logic in simplePar.py is going to create additional processes to be processed by the other cores on the machine.  This is different than the Random Search, where we intentionally created 12 different copies of the python file.

## Python File
```Python
import pandas as pd
from sklearn.preprocessing import StandardScaler
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score
from time import time

#This is the same as the dataset we used before, but
#about an order of magnitude bigger due to copying rows.
#This is to illustrate how these methods perform using
#reasonably large datasets.
data = pd.read_csv('studentpor_bigger.csv')

#Similar to the original grid search example,
#Here we'll be implementing a parallel job across
#a single node by increasing the number of threads
#we use.  We'll time things to show the difference
#between single and multi-core use.
#Note in the job script the -c 1 command!
#That tells the scheduler to only create *one*
#version of this script.  This script then handles
#the parallelization itself.


X = data[["traveltime", "studytime"]].values
y = data["Walc"]

scale_X = StandardScaler()
X = scale_X.fit_transform(X)


#This will be slow - one job only.
start = time()
rf = RandomForestClassifier(n_estimators=500, n_jobs=1)
rf.fit(X, y)
end = time()
res = end-start
print('%.3f seconds (1 core)' % res)
#For my run, n_jobs = 1 results in a 27.131 second runtime.


#Now we'll set this to 12 - this will create 12 processes.
start = time()
rf = RandomForestClassifier(n_estimators=500, n_jobs=12)
rf.fit(X, y)
end = time()
res = end-start
print('%.3f seconds (12 cores)' % res)
#For my run, this results in a 8.334 second runtime.
```

In this python file, you'll note the "n_jobs" option within each of the RandomForestClassifiers.  This determines the number of processes that are created, and then each process runs some number of the estimators (i.e., classification trees). 

## Submitting the files & Results
Once you've created your files, you can do a qsub <j> , where j is the name of your job file.  Make sure that your job file has the correct conda environment and .py file path specified, as they will likely be different than mine.
  
Once run, you should find a new output.out file in your directory.  Running a cat output.out, you should see a few things, including:
  
```
Thu Sep 08 2022 13:17:10 EDT: /usr/local/seoul/linux-centos7-piledriver/intel-18.0.5/mvapich2-2.3.1-m2no/bin/mpirun_rsh -rsh -np 1 -hostfile /tmp/mvp2run.hosts.28604  MV2_ENABLE_AFFINITY=1 MV2_SMP_USE_LIMIC2=0 MV2_SMP_USE_CMA=0 /sciclone/home20/dsmillerrunfol/.conda/envs/aml35/bin/python simplePar.py
```
 This string simply shows what time the job was processed, and the actual command that mvp2run created for you.  Until you get to much more complex models, you won't need this for debugging, but it is nice to have!
  
```
27.136 seconds (1 core)
8.407 seconds (12 cores)
```
  
Ignoring a few deprecation warnings, after that you will see the output from the test.  As you'll see, the multi-core processing is much faster, but not in a linear way.  As you introduce more cores (and, eventually, nodes), you'll see that intra-node communication and startup times are all non-trivial costs - so, you want to make sure your problem is big enough to warrant multiple cores before jumping in the deep end!
  
  

