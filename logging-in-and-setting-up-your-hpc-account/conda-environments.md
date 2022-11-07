# 🐍 Conda Environments

Setting up Conda environments on the cluster is very similar to how you would do it on your local computer. It is _highly recommended_ that you use Conda environments when running code the HPC.

To create a Conda environment and install packages, first log in to a frontend on SciClone with a terminal (i.e., ssh <yourname>@bora.sciclone.em.edu).  After you log in, copy these two lines into your terminal and hit enter.

```
source "/usr/local/anaconda3-2021.05/etc/profile.d/conda.csh"
module load anaconda3/2021.05
```

This will source and load the Anaconda module so it is ready for you to use.

Next, follow the standard steps for creating a Conda environment. Use the following line to create your environment and type `y` when prompted `Proceed ([y]/n)?`

```
conda create -n [ENVNAME]
```
(note some of the scikit learn examples assume your envname is "aml35" and you use python=3.5, i.e. `conda create -n "aml35" python=3.5`)

Next, activate your new environment by typing:

```
conda activate [ENVNAME]
```

To see a list of all the environments you've created, type:

```
conda info --envs
```

Once you've activated your environment, you can install any packages you need for a program using standard pip install commands. I.e.:

```
conda install pandas
```