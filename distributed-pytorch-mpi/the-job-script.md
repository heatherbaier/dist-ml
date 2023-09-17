# The Job Script



As mentioed int eh previous page, in order to performdistributed traiingusing PyTorch, we have to setup a little server that handles the traiing.This sserver iscomprisedof the master node and allother nodesinvolved in training.

In order to set this up, we have to run a line that looks likethis:

```shellscript
torchrun --nnodes=$NUMNODES --nproc_per_node=$PPN --rdzv_id=790876 --rdzv_backend=c10d --rdzv_endpoint=$IP_ADDRESS /sciclone/home20/hmbaier/tm/run.py $PPN $NUMNODES


```

on every node that willbe involved in training.

Todo so, we'll use a shell script that will launch a processonwhat willbe our masternode, grab the IP address of that node, save it, and usethat incormation to raunthe torchrun statement oevery other nodes that we want to involve in training.

```

#!/bin/bash

rm /sciclone/home20/hmbaier/tm/date.txt

qsub head_node.sh -l nodes=1:$2:ppn=$3 -v NUMNODES=$1,PPN=$3

# wait until the head node has launched and created the ip address file
while [ ! -f /sciclone/home20/hmbaier/tm/here.txt ]
do
  sleep 1
done

# grab the ip address from the file
value=$(</sciclone/home20/hmbaier/tm/here.txt)

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
