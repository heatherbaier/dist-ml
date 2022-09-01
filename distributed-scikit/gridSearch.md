# Grid Search
A grid search is the most obvious - and easiest - way to leverage distributed computation capabilities.  In a grid search, you are attempting to identify the best hyperparameters for a given model, but each model can be run independently.  Thus, we can spin up models across dozens of nodes without worrying about communications within a given job.
This is certainly not the only way you can use the HPC for machine learning (we'll continue to add examples to this guide as we generate more), but it's an excellent entry point to start to understand how multi-node computing works in the context of Machine Learning.

## Setting up your Python File
