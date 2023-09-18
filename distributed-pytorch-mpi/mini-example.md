# Mini Example

In this page, we're going to walk through what it looks like to actually set up a mini PyTorch training server. This should give you an idea of what it looks like when the nodes to connect during training.

First, open two terminals and log them into the same cluster. I am using Meltemi in this case.

ssh hmbaier@meltemi.sciclone.wm.edu

Then, submit and interactive job in each terminal, requesting 1 node and 12 processors for 1 hour on each.

```
qsub -I -l nodes=1:vortex:ppn=12,walltime=01:00:00
```

<figure><img src="../.gitbook/assets/Screenshot from 2023-09-18 13-42-31.png" alt=""><figcaption></figcaption></figure>

First, we need to figure out the IP address of our master nodes. It doesn't matter which node this is, so in either of your terminals, type:

```
hostname -i
```

This will give you the IP address of your node.

Next, type in&#x20;
