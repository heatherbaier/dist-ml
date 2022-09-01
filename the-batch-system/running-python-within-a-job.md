# Running Python within a job

Running a python script within a job requires a few lines of setup. Just like when we created our Conda environments, we need to actually source and load the Python language. On Vortex, we'll do this using this line:

```
unsetenv PYTHONPATH
```

So now, if we combine that with the job script from the previous page and the Conda page, we now have:

```
!/bin/tcsh
#PBS -N demojob
#PBS -l nodes=1:vortex:ppn=12
#PBS -l walltime=00:30:00

source "/usr/local/anaconda3-2021.05/etc/profile.d/conda.csh"
module load anaconda3/2021.05

unsetenv PYTHONPATH
```

So now, let's create a simple Python script that print's Hello World! We'll make it in the same demo folder that we created on the previous page. Once you are in your demo folder (check the previous page for instructions), run the following lines to create your Python script and open it with the nano editor:

```
touch hello.py
nano hello.py
```

Once in the nano editor, copy in the line below and exit the editor.

```
print("Hello world!")
```

Now, open your old job script in the nano editor using `nano job` and match it to the below code:

```
!/bin/tcsh
#PBS -N demojob
#PBS -l nodes=1:vortex:ppn=12
#PBS -l walltime=00:30:00

source "/usr/local/anaconda3-2021.05/etc/profile.d/conda.csh"
module load anaconda3/2021.05

unsetenv PYTHONPATH

cd demo
python hello.py
```

Note: Every job starts a terminal within your home20 directory no matter which directory you launch the job script from, so before we run our Python script, we need to cd into our demo directory within the job script.

Now, run `qsub job` and wait for your job to finish and your output files to be created. Open the file labeled `job.o....` and the last line should say Hello World!

# Other notes and approaches

Depending on your specific configuration, needs and goals, you may want to initialize your job scripts and python files in different ways than what we talked about above.  For example, you may want more granular control over your outputs, more explicit declarations of conda envioronments, or other changes.  Also of note, the HPC at W&M is constantly being upgraded, and so some solutions (such as the unsetenv PYTHONPATH above) may no longer be necessary.  As an example of some alternative approaches you can implement, consider the below job and python file:

``` job file
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

```python file
import sys
print(sys.version)
```

If you qsub the above files, you will be provided with a single output (output.out) that contains the outputs of hello.py.  In this example, if the conda environment aml35 has a python 3.5 installation active, it should report back that the version of Python is 3.5 when it runs on the cluster.  You can also manually create files from within your python script, just like you would on a local machine.
