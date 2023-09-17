# Random Search - Simple

A random search is the most obvious - and easiest - way to leverage distributed computation capabilities. In a random search, you are attempting to identify the best hyperparameters for a given model, but each model can be run independently. Thus, we can spin up models across dozens of nodes without worrying about communications within a given job. This is certainly not the best way you can use the HPC for machine learning, but it's an excellent entry point to start to understand how multi-node computing works in the context of Machine Learning.

## Setting up your Conda environment

We're going to need a few things for our conda environment (which should be using py3.5):

conda activate aml35

conda install scikit-learn==0.19.2

conda install pandas==0.23.4

## Setting up your Python File

There are a number of ways to launch a python file on the cluster - for example, you can have a "main" script which tells workers when to run, submits it's own jobs, or just handles communications. Alternatively, you can run the same script across all nodes, using unique identifiers to ensure you don't run into conflicts (i.e., two nodes trying to write to the same file at the same time).

For this example, we'll take the simplest road which is to have the same python file launch on every node for our grid search. This is fairly straightforward - we're just going to load in a scikit-learn model and run it, with the caveat that we're going to output a unique file for each node that contains the outputs we care most about.

First, we'll have our python file:

```
import pandas as pd
from sklearn.preprocessing import StandardScaler
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import accuracy_score
import random

data = pd.read_csv('studentpor.csv')

#We'll run a model that attempts to predict if a student
#if a student has very low (0) or very high (5) weekend
#alcohol consumption (Walc) based on their studytime weekly
#(studytime) and traveltime to school (traveltime).
#For this example, we'll be treating studytime
#and traveltime as continious variables, which isn't
#perfectly accurate, but close enough. We aren't really
#trying to get the best model we can, but rather illustrate
#a distribution strategy.  We're also not doing lots of other
#things we might want to, like split our data, etc.

X = data[["traveltime", "studytime"]].values
y = data["Walc"]

scale_X = StandardScaler()
X = scale_X.fit_transform(X)

#C is our regularization strength - larger values mean
#weaker regularization.
#We'll do a random search here, from 0 to 10.
C = random.random() * 10
logC = LogisticRegression(penalty="elasticnet", solver="saga", fit_intercept=False, tol=1.0, C=1.0)
logC.fit(X, y)

#Percent accuracy:
acc = accuracy_score(y, logC.predict(X))

#Save it into a file with our C:
f = open("/sciclone/home20/dsmillerrunfol/dml/results/" + str(C)+ ".csv", "w")
f.write(str(C) + "," + str(acc) + "\n")
f.close()

#Once we're done, we should have a folder full of csv
#files.  All we need to do is concatenate them together
#into one output: cat *csv > all.csv
```

And, second, our job file:

```
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
mvp2run -c 24 python distributeSearch.py >& output.out
```

A few important things to note here. First, we now have 2 nodes with 12 processors each specified - so, 24 total processors.

Second, we now have mvapich2-ib loaded. We'll use that to get mvp2run to work, which is what we use to tell the cluster to create 24 different copies of our python file, one on each node.

Finally, at the bottom you'll see we now call mvp2run - the -c 24 says to spin up 24 total copies of our python file. With all of that, you'll get 24 outputs in your /results/ file, one for each run. You can concatenate the ouputs with:

`*csv > all.csv`
