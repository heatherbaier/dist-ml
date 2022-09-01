# Grid Search
A grid search is the most obvious - and easiest - way to leverage distributed computation capabilities.  In a grid search, you are attempting to identify the best hyperparameters for a given model, but each model can be run independently.  Thus, we can spin up models across dozens of nodes without worrying about communications within a given job.
This is certainly not the only way you can use the HPC for machine learning (we'll continue to add examples to this guide as we generate more), but it's an excellent entry point to start to understand how multi-node computing works in the context of Machine Learning.

## Setting up your Conda environment
We're going to need a few things for our conda environment (which should be using py3.5):
conda activate aml35
conda install scikit-learn==0.19.2
conda install pandas==0.23.4

## Setting up your Python File
There are a number of ways to launch a python file on the cluster - for example, you can have a "main" script which tells workers when to run, submits it's own jobs, or just handles communications.  Alternatively, you can run the same script across all nodes, using unique identifiers to ensure you don't run into conflicts (i.e., two nodes trying to write to the same file at the same time).
For this example, we'll take the simplest road which is to have the same python file launch on every node for our grid search.  This is fairly straightforward - we're just going to load in a scikit-learn model and run it, with the caveat that we're going to output a unique file for each node that contains the outputs we care most about.

