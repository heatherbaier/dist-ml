# Introduction

Distributed computing and parallel computing have one key difference: parallel computing simultaneously uses all of the resources available on a single computer to perform tasks concurrently. Distributed computing simultaneously uses the resources across a cluster of computers to perform tasks simultaneously. There are two key advantages of this.

First, when using very large datasets to train deep learning models, by distributed our dataset across multiple machines, we increase the memory available during training. This is one of the two core types of distributed learning, called **Distributed Data Parallel.**

The second key advantage is when you are training a model that requires a large memory footprint. This is the second type of distributed training, called **Distributed Model Parallel**. In this case, we would programmatically send set sets of layers in our model to different machines within our cluster, then use a technique called _distributed autograd_ to preform our back-propagation on weight update. More on this later!
