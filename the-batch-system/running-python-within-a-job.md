# Python + Conda in a Job

Using python with anaconda on the cluster is nearly identical to how you use it on your frontend to configure environments.

For example, if you follow the instructions in the section on creating [Conda Environments](https://hmbaier.gitbook.io/distributed-ml-w-and-m/logging-in-and-setting-up-your-hpc-account/conda-environments) on the node you log in on, when you [submit jobs](https://hmbaier.gitbook.io/distributed-ml-w-and-m/the-batch-system/non-interactive-jobs) you can then use the same environments like this:

```
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

cd dml
python hello.py >& output.out
```

```python
import sys
print(sys.version)
```

If you qsub the above files, you will be provided with a single output (output.out) that contains the outputs of hello.py. In this example, if the conda environment aml35 has a python 3.5 installation active, it should report back that the version of Python is 3.5 when it runs on the cluster. You can also manually create files from within your python script, just like you would on a local machine.
