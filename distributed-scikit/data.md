# Example Dataset

To illustrate how to use sklearn in a distributed system, we'll be using a small CSV about student alcohol consumption off of Kaggle (see https://www.kaggle.com/datasets/uciml/student-alcohol-consumption).  As a first step, you'll need to upload the CSV into your home directory on SciClone (or any other directory you may want).  The easiest way to do this with a GUI is through a program like Filezilla (a free file transfer program); you can also use terminal-based tool such as `scp` if you are familiar with them.

To upload the file with a program such as filezilla, you'll need to use a SFTP-SSH file transfer protocol, and point your host to bora.sciclone.wm.edu.  You'll enter your username and password, and then be provided with a fairly straightforward interface for file uploads - you can also see the full [Filezilla tutorial](https://hmbaier.gitbook.io/distributed-ml-w-and-m/logging-in-and-setting-up-your-hpc-account/filezilla)

![image](https://user-images.githubusercontent.com/7882645/187920024-f70adab3-b7ce-45c5-a2e7-8c199929229e.png)

From here, you can just drag and drop the file you want to upload - in our case, we'll be uploading the "studentpor.csv" file.  Once you've uploaded it, you can confirm it is present by typing `ls` into your sciclone terminal, i.e.:

![image](https://user-images.githubusercontent.com/7882645/187920286-16124efd-afeb-4665-8a0f-d1dbdf3db5a1.png)