# The workers file



While you won;t ever directly run the workers.sh file becuase it's executed by run.sh, it's important to know what's going on in it. For the most part it looks like a normal job script (setting up the environment, loading conda and Python and activating your environment), the last line is the toruchrun line we discussed in the last page. This command line argument accepts the PPN, NUMNODES and IP address arguments we initialized in run.sh in order to connect to the torch training server form our former nodes.

```
#!/bin/tcsh
#PBS -N worker
#PBS -l walltime=24:00:00
#PBS -j oe

# init conda within new shell for job
source "/usr/local/anaconda3-2021.05/etc/profile.d/conda.csh"
module load anaconda3/2021.05
unsetenv PYTHONPATH
conda activate dhsrl4

echo $IP_ADDRESS

torchrun --nnodes=$NUMNODES --nproc_per_node=$PPN --rdzv_id=790876 --rdzv_backend=c10d --rdzv_endpoint=$IP_ADDRESS /sciclone/home20/hmbaier/tm/run.py $PPN $NUMNODES

```
