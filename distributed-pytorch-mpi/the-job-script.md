# The Job Script

As mentioned int eh previous page, in order to perform distributed training using PyTorch, we have to setup a little server that handles the training. This server is comprised of the master node and all other nodes involved in training.

In order to set this up, we have to run a line that looks like this:

```shellscript
torchrun --nnodes=$NUMNODES --nproc_per_node=$PPN --rdzv_id=790876 --rdzv_backend=c10d --rdzv_endpoint=$IP_ADDRESS /sciclone/home20/hmbaier/tm/run.py $PPN $NUMNODES
```

on every node that will be involved in training.

To do so, we'll use a shell script that will launch a process on what will be our master node, grab the IP address of that node, save it, and use that information to run the torchrun statement on every other node that we want to involve in training. The job script looks like this:

```

#!/bin/bash

rm /sciclone/home/hmbaier/tm/date.txt

qsub head_node.sh -l nodes=1:$2:ppn=$3 -v NUMNODES=$1,PPN=$3

# wait until the head node has launched and created the ip address file
while [ ! -f /sciclone/home20/hmbaier/tm/here.txt ]
do
  sleep 1
done

# grab the ip address from the file
value=$(</sciclone/home/hmbaier/tm/here.txt)

echo $value
echo $2
echo $3

rm /sciclone/home20/hmbaier/tm/here.txt

# then submit the job array with the ip address
for ((i = 1; i <= $1-1; i++))
do
  qsub workers.sh -l nodes=1:$2:ppn=$3 -v IP_ADDRESS=$value,NUMNODES=$1,PPN=$3
done
```

We run the job script with 2 arguments. The first is the number of nodes we want to use in training, the second is the cluster we are using (vortex, bora or meltemi normally) and the third is the number of processors we want per node.

We run the qsub by executing, for example:

```
run.sh 2 meltemi 12
```

This would launch a training server on Meltemi with 2 nodes and 12 processors per node.
