# MPI & Python

### What is MPI?

MPI stands for Message Passing Interface and is a way of passing messages between multiple computers running a parallel program.

> **Rank:** Each process in a parallel program has a unique rank, i.e. an integer identifier. The rank value is between 0 and the number of processes - 1.

The sum of all of these ranks is called the _world size_.

> **World Size:** Total number of ranks in a parallel program.

### Types of Communication

> **Blocking Communication:** Blocking communication halts all activity within a process until a message has been sent or received

> **Non-Blocking Communication:** Non-blocking communication allows activity within a process to continue, even before a message has been fully set or received.

#### Point to Point (P2P)

> **Point-to-Point (P2P) communication:** When only two processes within a parallel program communicate with each other.&#x20;

For example, there exists a parallel program that has 10 processes (i.e. the world size is 10). Process 1 sends a message to Process 2 letting it know that it has completed a task, but does not communicate with any other process. This is an example of P2P communication.

_Blocking P2P Methods_

send...

recv...

_Non-Blocking P2P Methods_

isend...

irecv...

#### Collective Communication (CC)

> **Collective communication (CC)**: A type of communication that involves all or some of the processes in a parallel program.

For example, we take our same parallel program that has 10 processes. The objective of process 1 is to average a value that is stored on each of the other processes. In this case.process 1 would issue a CC call to the other nodes and _collect_ their values in order to average them. Because process 1 communicated with multiple nodes instead of just 1, this is an example of collective communication.

_CC Methods_

bcast

scatter

gather

#### References & Helpful Sites
