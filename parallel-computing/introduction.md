# Introduction

Parallel computing works by launching multiple processes on a computer that run in parallel. You can run parallel programs in Python using a few different methods, but we'll go over two.&#x20;

[The first method](multiprocessing.md) is the most simple and uses a Python package called _multiprocessing._ This method can be used on any computer that has more than one processor (yes, this includes your personal laptop). This package allows you to do things like parallelizing a for loop and can be super helpful in everyday programming.

We enter the world of MPI (Message Passing Interface) in [the second method](mpi-and-python.md). This is a more involved method than the first and requires setting up a 'world' and assigning ranks to each of the processes that will be involved in our program.
