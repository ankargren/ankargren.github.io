---
layout: post
title: Running simulations in R on UPPMAX
category: R

excerpt: A brief guide explaining how to set up a simulation in parallel on UPPMAX, discussing some mistakes I made at first and how to avoid them.
---

In this post, I will briefly go through the steps needed in order to run a simulation in R (or any parallel job) on UPPMAX. I started writing it for my own sake so I wouldn't forget how I did it, but I thought that I'd share it so that others could possibly avoid making the same mistakes I did.

First of all, you need an account and a project which has been granted access to the servers. You also need to set it up so that you can log on to the system. This is described [in the official UPPMAX guide](https://www.uppmax.uu.se/support/user-guides/guide--first-login-to-uppmax/). In the following, my username is `sebaa` and `snic2015-6-117` is my project name.

When you have it all set up, that's when the fun part begins. Essentially, if you have a parallel job already set up in R, the process consists of the following steps:

1. Prepare the servers for your job
2. Write a bash script, which tells the system how much power you need
3. Run the bash script

Step 1 means installing necessary packages, creating directories if needed, etc. Step 2 includes preparing your computational request, so that you get the desired resources which your computations in R will use. Step 3 merely amounts to letting it all run (including a call to your R code).

## Installing packages
R is installed on the servers, but if your script relies on external packages you'll have to install these before running anything. To check whether e.g. the `lavaan` package is installed, you can enter R and try to load it:

    [sebaa@rackham3 ~]$ R 
    > require("lavaan")
    Loading required package: lavaan
    Warning message:
    In library(package, lib.loc = lib.loc, character.only = TRUE, logical.return = TRUE,  :
      there is no package called ‘lavaan’

To install the packages you need, you can either do it from the terminal (e.g. `install.packages("lavaan", repos = "http://cloud.r-project.org/")`), or you can open up RStudio and do it interactively. For the latter, first close down R and then open RStudio as follows:

    > quit()
    [sebaa@rackham3 ~]$ rstudio

Note here that `>` means you're in the R console, whereas `[sebaa@rackham3 ~]$` means you're in the usual (Ubuntu) terminal. 

## Small (single-node) jobs
The Rackham cluster consists of nodes with 20 cores each. In order to set up the parallel computation, you need to decide on what amount of resources to request. If you request 20 cores or less (i.e. one node or less), you can use a `PSOCK` cluster in the `parallel` package directly---this would, in many cases, be what would be used if only using a local machine. If so, you can create your cluster in R using 

    cl <- makeCluster(n_cores, type = "PSOCK")

where `n_cores` is the number of cores requested (not exceeding 20). 


## Large (multiple-node) jobs
My first mistake was to request 5 nodes (100 cores), but using the `PSOCK` way of creating the cluster in R. The reason why this doesn't work is that it only makes use of one of the nodes, so I was using only one node but paying for five. It is possible to circumvent this by explicitly creating the workers on the different nodes you have been granted access to; however, when I tried this there was a bug in the between-node communication.

Instead, what I chose to do is use OpenMPI and the `Rmpi` package together with `snow`. 

### Installing and using Rmpi
To get started with this, you first need to install `Rmpi`. To do that, you need to load OpenMPI and to do this you need to load a specific version of `gcc`, which is the compiler. Run the following on the UPPMAX server:

    [sebaa@rackham3 ~]$ module load R/3.3.2
    [sebaa@rackham3 ~]$ module load gcc/7.1.0
    [sebaa@rackham3 ~]$ module load openmpi/2.1.1

followed by

    [sebaa@rackham3 ~]$ R
    > install.packages("Rmpi", repos = "http://cloud.r-project.org/")`

After this, you should have the `Rmpi` package installed!

The process of creating an MPI cluster is now simple: create the cluster outside of R using `mpirun` and retrieve the cluster in R using `snow::getMPIcluster()`. 

## Interactive example
To test things out, we can create an interactive development session. The following asks for an interactive session for the project `snic2015-6-117` (this is the `-A` option) with two nodes (`-n 40`) for 15 minutes (`-t 15:00`). The `-p devel` and `--qos=short` options give us higher priority in the queue, but also limit us to just a few nodes (at most) and at most an hour of CPU time.

    [sebaa@rackham3 ~]$ interactive -A snic2015-6-117 -p devel --qos=short -n 40 -t 15:00

After this, a session should be initiated. You will also see that you are moved from one of the four login nodes (`rackham1`--`rackham4`) to a compute server (a node, `r1`--`r334`).

Now, load R and OpenMPI as before:

    [sebaa@r331 ~]$ module load R/3.3.2
    [sebaa@r331 ~]$ module load gcc/7.1.0
    [sebaa@r331 ~]$ module load openmpi/2.1.1

When creating the MPI cluster, we also launch R using the shell script `RMPISNOW` as described [here](http://homepage.divms.uiowa.edu/~luke/R/cluster/cluster.html). This requires you to have `RMPISNOW` on your path; most likely, you will find this in `user/R/.../`. Add it to `PATH`:

    export PATH=$PATH:/home/sebaa/R/x86_64-pc-linux-gnu-library/3.3/snow/

The above command redefines `PATH` to be its old value (`$PATH`) plus the new path. You can verify that it worked by running `echo $PATH` (it should now be at the end). 
Next, to launch R we run

    [sebaa@r331 ~]$ mpirun -np 40 RMPISNOW

Remember that we previously asked for 40 cores. Here, we are starting an MPI job using all of the 40 cores (the `-np 40` option). 

You'll see now that R is launched. To retrieve the MPI cluster inside R, you run

    cl <- snow::getMPIcluster()

and after that `cl` is your cluster, which you can use together with the `par*apply()` functions, `clusterExport()`, etc, as usual. For example, to compute the mean of 10,000 normal random variables 40 times you'd call

    clusterCall(cl, mean(rnorm(10000)))

To kill your interactive session prematurely, you can get the `JOBID` by running
`jobinfo -u sebaa`. If the ID of your job is 538417, you can cancel it by `scancel 538417`.

## Batch file
The interactive session is great for testing things, but for real jobs you probably want to put it all in a script. First, we create the R script and save it as `rscript.R`

    library(Rmpi)
    library(snow)
    cl <- getMPIcluster()
    res <- clusterCall(cl, mean(rnorm(10000)))
    save(res, file = "res.RData")

Put this in your user root (home directory). 
As for the bash script, this will basically just be a collection of the commands we previously issued---from the resource request to the loading of modules and creation of MPI cluster. Here is what it would look like:

    #!/bin/bash -l
    #SBATCH -A snic2015-6-117
    #SBATCH -p devel --qos=short
    #SBATCH -n 40
    #SBATCH -t 15:00:00
    module load R/3.3.2
    module load gcc/7.1.0
    module load openmpi/2.1.1
    export PATH=$PATH:/home/sebaa/R/x86_64-pc-linux-gnu-library/3.3/snow/
    mpirun -np 40 RMPISNOW --no-save < rscript.R

And that is all there is to it! For real jobs, you want to replace `#SBATCH -p devel --qos=short` with `#SBATCH -p node` (or `-p core`) and maybe also change the number of requested workers to something other than 40. To assess your job, there is [jobstats](https://www.uppmax.uu.se/support/user-guides/jobstats-user-guide/).

