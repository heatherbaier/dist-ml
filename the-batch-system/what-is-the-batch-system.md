# 👩🌾 What is the batch system?

Think of the HPC like a community garden - users reserve individual plots within a garden space so that people can't plant in their area and they have room to grow what they wish. Think of a whole sub-cluster (i.e. Vortex) as the garden and each of the nodes as a plot you can reserve. By submitting jobs, we reserve the resources on the nodes for ourselves. If we were to run python scripts, programs, etc.. without submitting jobs, we would all constantly be fighting for resources and our regularly collide with each other. This is what would happen if you were to log into the HPC and run a python program outside of a job - you would essentially be using resources that you did not fairly check out (i.e. you would be planting your veggies in everyone else's garden plots).

### How the Job Queue works

When you submit a job (interactive or non-interactive), you **must** specify a few attributes regarding what type of computer node(s) you want to use for your analysis.  These include the number of nodes, the cluster (i.e., what computer architecture - including memory - you want), the length you want the resources for (hours, minutes and seconds), and the number of processors you need on the node. 

Based on these factors, you are entered into a queue - if you request relatively few resources for short periods, the system aspires to give you resources as quickly as possible; jobs with larger walltimes and higher numbers of resources may take a long time before they are able to run.  The specific scheduler we use (Maui) seeks to optimize the cluster so that as many core-hours as is feasible can be provided across all requests.

The take-away from this is that it is important to not request more than you need - i.e., launching more, smaller jobs will enable more flexibility in scheduling, and thus provide you with a higher likelihood of receiving resources.

#### Basic Commands to Understand the Job Queue

If you are not getting scheduled, there are a few basic commands you can try to debug the issue. A few helpful options include:

`qstat` - Provides a list of all jobs currently in the system, requested times, status ("R" for running, "Q" for in queue) and other factors.

`qstat -u <username>` - I.e., "qstat -u dsmillerrunfol".  Provides a list of all of your current jobs - and `jobIDs` - currently in the system.

`showstats` - Provides a synopsis of cluster-wide availability.

`pbstop` - Shows specific use of clusters and nodes at a given time.

`checkjob <jobID>` - Show detailed information on what the job has requested, and some debug information regarding why a job hasn't started (or, if it's running, run statistics).

`showstart` - Shows the systems estimate for when a job will start (or when it started, if already underway).  

`showq` - Shows the full queue of jobs waiting to be scheduled.  `showq - u <username>` shows your queue.

####
