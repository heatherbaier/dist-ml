# Basics

Distributed training involves distributing a model across multiple nodes on a cluster. Wemight do this fora few reasons:

1. When our model is too large to fit on a single node
2. When our dataset is large and training is slow, we distribute our dataset across multiple nodes in order train the model in parallel.

In our case, we are distributing for the second reason. We have a large dataset and training is very slow. Therefore, we are going to split the dataset into N pieces (with N being the number of nodes \* processors). From there, we are going to copy our model onto each node. When we start training, every node has a different partition of the dataset, and batch 1 in epoch 1 will look differnt on each node. On every batch, we will do a forward pass on each node, calculate the loss andgradients,thensendthose gradients tothe masternode. The master node will then average all of those gradients, update the model's weights, and distribute those back to the models on each node.
