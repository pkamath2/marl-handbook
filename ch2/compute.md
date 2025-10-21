# A Really Short Getting Started Guide with HPC for MARL

While much of the day-to-day work we do in the lab can be done on a personal computer, our primary resource
for computing is the [High-Performance Computing (HPC) cluster](https://sites.google.com/nyu.edu/nyu-hpc).
The HPC can be used for large distributed jobs, but also provides a notebook server for a more interactive
workflow that you may be used to.


This document is a quick start guide for getting set up and accessing NYU HPC (high performance compute) infrastructure. It is not meant to be a comprehensive document, but something to get you started with working with HPC if you are new to NYU (and MARL). Please read the documents listed below for a more comprehensive understanding of the compute infrastructure. 


## Table Of Contents
1. [Accessing HPC and First Time Login](#Accessing-HPC-and-First-Time-Login)
2. [Singularity Containers and Overlays](#singularity-containers-and-overlays)
3. [Using HPC (singuconda and SLURM)](#using-hpc-singuconda-and-slurm-to-run-your-code)
4. [Hugging Face Hacks](#hugging-face-hacks)
5. [Using VSCode to debug code using GPU Cluster](#using-vscode-to-debug-code-using-gpu-cluster)


**Note**: Please skim through these important links/documents before reading this "short getting started guide". Most of these link might be available only on NYU network via campus WiFi or VPN.   

[1] Accessing HPC: https://sites.google.com/nyu.edu/nyu-hpc/accessing-hpc?authuser=0   
[2] Getting a new account: https://sites.google.com/nyu.edu/nyu-hpc/accessing-hpc/getting-and-renewing-an-account?authuser=0  
[3] Raise a new access request: https://identity.it.nyu.edu/identityiq/home.jsf  
[4] Conda and Singularity - 1: https://sites.google.com/nyu.edu/nyu-hpc/hpc-systems/greene/software/open-ondemand-ood-with-condasingularity?authuser=0   
[5] Conda and Singularity - 2: https://sites.google.com/nyu.edu/nyu-hpc/hpc-systems/greene/software/singularity-with-miniconda   
[6] Singuconda - https://github.com/beasteers/singuconda/tree/main
[7] NYU HPC GPU List - https://sites.google.com/nyu.edu/nyu-hpc/hpc-systems/greene/hardware-specs   
[8] HPC Diskspace Hardware Specs - https://sites.google.com/nyu.edu/nyu-hpc/hpc-systems/hpc-storage/hardware-specs    



## Accessing HPC and First Time Login

* Raise a request to get access to HPC using this link - https://identity.it.nyu.edu/identityiq/home.jsf. This step creates your Unix id and a user profile on HPC. You will need to be on the NYU network (either via campus WiFi or VPN) to access the link.  

    On the request webpage, go to Side bar -> Manage Access -> Request HPC Account. Fill in the necessary details. Your request should be complete once your supervisor/faculty approves your request (this takes approximately a few hours).   
  
    For more details see [[2]](https://sites.google.com/nyu.edu/nyu-hpc/accessing-hpc/getting-and-renewing-an-account?authuser=0) in the document links above. 

* Once you receive an email that access is complete, update SSH Config on your local machine. Replace `<Net ID>` with your NYU Net ID.
    For Mac or Linux, update ~/.ssh/config  with the following: 
    ```
    Host greene.hpc.nyu.edu dtn.hpc.nyu.edu gw.hpc.nyu.edu
      StrictHostKeyChecking no
      ServerAliveInterval 60
      ForwardAgent yes
      UserKnownHostsFile /dev/null
      LogLevel ERROR
      User <Net ID>
    ```
    
    For Windows and other SSH Config options please see [[1]](https://sites.google.com/nyu.edu/nyu-hpc/accessing-hpc?authuser=0 ) in the document links above.
    
* Next, you can logon to the HPC login nodes using yout Net ID and password using iTerm (on Mac) or PuTTy (on Windows)- 
    ```ssh <Net ID>@greene.hpc.nyu.edu```  

    **Note:** If you are not on the NYU network, you will need to connect to VPN. 

    Also, see how to set up SSH Keys on your local machine to not have to use your Net ID password on every login in the document at [[1]](https://sites.google.com/nyu.edu/nyu-hpc/accessing-hpc?authuser=0 )

* Your home should be under `/home/<Net ID>`. Run `myquota` to see your disk space allocations. For detailed explanations on the type and use of each of these disks please see the HPC hardware specs at [[8]](https://sites.google.com/nyu.edu/nyu-hpc/hpc-systems/hpc-storage/hardware-specs). 

    ```
    [<your Net ID>@log-2 ~]$ myquota
    
    Hostname: log-2 at Thu Sep 11 02:42:29 PM EDT 2025
    
    Filesystem   Environment   Backed up?   Allocation       Current Usage
    Space        Variable      /Flushed?    Space / Files    Space(%) / Files(%)
    
    /home        $HOME         Yes/No       50.0GB/30.0K       0.01GB(0.01%)/15(0.05%)
    /scratch     $SCRATCH      No/Yes        5.0TB/1.0M       0.00GB(0.00%)/1(0.00%)
    /archive     $ARCHIVE      Yes/No        2.0TB/20.0K       0.00GB(0.00%)/1(0.00%)
    /vast        $VAST         NO/YES        2TB/5.0M           0.0TB(0.0%)/1(0%)
    ```

    We typically use `/home` for small projects and codebases. `/scratch` is much larger and writable, so many of us host larger projects under a directory in `/scratch`. 

    Please also reach to your supervisor (or Brian McFee) for access to the MARL datasets directory under `/scratch`. 

    Also, make a note of the limit on the number of files you can save under each filesystem, and not just the space. HPC diskspace is better at accomodating a fewer large files than multiple smaller files.   


* Create some symlinks inside your home directory to /scratch and /vast folders for ease of access.  

    ```
    ls /scratch/<Net ID> ## This should exist for your Net ID. If not, please raise a support request.  
    cd ~ 
    ln -s /scratch/<Net ID> scratch  
    ln -s /vast/<Net ID> vast
    ```
 

    You will notice that NYU HPC has three login CPU compute nodes/hosts, namely `log-1`, `log-2`, and `log-3`. Note that these are not GPU nodes.  
    
    **DO NOT RUN** any code on these login nodes directly. HPC admin will kill all long running jobs on login nodes. Always use SLURM (Or Singuconda as we will discuss shortly) to submit jobs to the CPU or GPU clusters. 
        
    Your home directory will be mounted on all three (i.e., can be accessed from all 3 nodes). Additionally, you will have access to a few other shared directories (such as `/scratch` and `/vast`) to store your code and data. 




## Singularity Containers and Overlays

NYU HPC uses Singularity for containerization. If you are familiar with containers such as Docker, Singularity should not be very difficult to understand and work with. However, there are some differences in the way you download and run Singularity containers on HPC. 

Firstly, the Singularity containers made available on HPC are read-only. These are referred to as SIF (or Singularity Image Format) files. This is by design, as everyone at NYU uses these containers, making them read-only prevents them from corruption. 

You can personalize a container (e.g., install libraries or write something to the file system), using an Overlay File System. This Overlay is a transparent layer that sits "on the top of" the read-only container. This file system can be simple directories, or most commonly it is an ext3 filesystem image (we use ext3 in the examples below). When the container is shut down, the Overlay details (installations and files) are saved separately from the container on the users file system as an ext3 image. 

Downloading Singularity container and an Overlay is a multistep process. If you are interested you can look at documents [[4]](https://sites.google.com/nyu.edu/nyu-hpc/hpc-systems/greene/software/open-ondemand-ood-with-condasingularity?authuser=0) and [[5]](https://sites.google.com/nyu.edu/nyu-hpc/hpc-systems/greene/software/singularity-with-miniconda) listed above.


Our friends at MARL (specifically [Bea Steers](https://github.com/beasteers)) have created a user friendly library to help us ease into the installation process. Singuconda (https://github.com/beasteers/singuconda/tree/main) assists us in installing and running Miniconda over Singularity Overlays. For more details head over to the projects Github page. 

For simple installation steps see below - 

### First time installation of singuconda (performed only once)

Log in to one of the login nodes and in your home directory and run:
```
curl -L https://github.com/beasteers/singuconda/raw/main/singuconda --output ~/singuconda
chmod +x ~/singuconda
```

### Subsequently, for every new project

* Create a project directory under `$HOME/scratch`. Lets call it 'myproject'. Run - 
    ```
    cd $HOME/scratch/myproject
    ~/singuconda
    ```
    Answer a few prompts on the size of your container (e.g., disk size and number of files you expect to save etc., e.g., 15GB; 500K - See example below)

    (Use your arrow keys to scroll down the overlay options. I typically use 15GB-500K overlay as selected below).
    
    ```
    [pk3251@log-1 scratch]$ pwd
    /home/pk3251/scratch
    [pk3251@log-1 scratch]$ mkdir myproject
    [pk3251@log-1 scratch]$ ~/singuconda

    You probably want to use either 5GB or 10GB.
    Any smaller and you will probably run out of space.
    Search: █
    ? Which overlay to use? (e.g. type '-5gb'):
        /scratch/work/public/overlay-fs-ext3/overlay-5GB-200K.ext3.gz
        /scratch/work/public/overlay-fs-ext3/overlay-10GB-400K.ext3.gz
    ▸ /scratch/work/public/overlay-fs-ext3/overlay-15GB-500K.ext3.gz
        /scratch/work/public/overlay-fs-ext3/overlay-3GB-200K.ext3.gz
    ↓   /scratch/work/public/overlay-fs-ext3/overlay-5GB-3.2M.ext3.gz


    ```

    Give it a name:

    ```
    [pk3251@log-1 scratch]$ ~/singuconda
    You probably want to use either 5GB or 10GB.
    Any smaller and you will probably run out of space.
    ✔ /scratch/work/public/overlay-fs-ext3/overlay-15GB-500K.ext3.gz
    ✔ Why don't you give your overlay a name?: myproject-overlay-15GB-500K█

    ```

    Choose an appropriate Singularity Image File (SIF) based on your Ubuntu versioning, CUDA version requirements etc.

    I chose Cuda12 below. 

    ```
    You choose "myproject-overlay-15GB-500K"
    Unzipping /scratch/work/public/overlay-fs-ext3/overlay-15GB-500K.ext3.gz to myproject-overlay-15GB-500K.ext3...
    Done!
    Search: █
    ? Which sif to use? (e.g. type 'cuda12'):
    ▸ /scratch/work/public/singularity/cuda12.3.2-cudnn9.0.0-ubuntu-22.04.4.sif
        /scratch/work/public/singularity/colabfold-1.5.5-cuda12.2.2.sif
        /scratch/work/public/singularity/cuda10.0-cudnn7-devel-ubuntu18.04-20201207.sif
        /scratch/work/public/singularity/cuda10.0-cudnn7-devel-ubuntu18.04.sif
    ↓   /scratch/work/public/singularity/cuda10.1-cudnn7-devel-ubuntu18.04-20201207.sif

    ```

    Remember to use the right python version:

    ```
    /ext3/miniconda3/bin/python
    You're currently setup with:
    Python 3.13.5
    To keep this version: Leave blank.
    Want a different python version? (e.g. 3.10, 3.12.1): 3.10

    ```
      
    The script should complete with the following logs. You are all set! 
    
    ```
    Great you're all set!

    To enter the container, run: ./sing

    or you can run:

    singularity exec  \
        --overlay myproject-overlay-15GB-500K.ext3:ro \
        /scratch/work/public/singularity/cuda12.3.2-cudnn9.0.0-ubuntu-22.04.4.sif \
        /bin/bash --init-file /ext3/env

    The above command opens with read-only. To open with write permissions: ./singrw

    ✔ nothing, byeee!

    Happy training! :)

    Quick commands: ./sing (read-only)    ./singrw (read-write)
    [pk3251@log-1 scratch]$
    ```

Next we will look at using SLURM to request GPU resources in an interactive mode. 

Run `./singrw` for read-write access to your overlay and `./sing` for read-only. Read-write access is needed when you want to create a new conda environment, install python libraries/packages, etc.

    


## Using HPC (singuconda and SLURM) to run your code

NYU HPC uses SLURM to submit a job to the CPU or GPU clusters. A job can be a batch (e.g., training your neural network on GPUs as a long running background thread/process), or an interactive job (e.g., running a Jupyter notebook)


**Note**: You will need a project account code to run any SLURM commands (on the CPU or GPU clusters). This code is of the format "pr_###_<function>". Your project PI/Supervisor should be able to create a new project on the HPC management portal and give you a code. 


### Running an interactive job
A quick example to submit an interactive job to the **CPU** cluster: 

``srun --cpus-per-task=12 --mem=32GB --time=2:00:00 --account=pr_###_function --pty /bin/bash``   

A quick example to submit an interactive job to the **GPU** cluster:

``srun --cpus-per-task=12 --gres=gpu --mem=32GB --time=2:00:00 --account=pr_###_function --pty /bin/bash``  

Specify number of GPUs you need: 
``srun --cpus-per-task=12 --gres=gpu:2 --mem=32GB --time=2:00:00 --account=pr_###_function --pty /bin/bash``

Specify the GPU you want to run this job on (see [[7]](https://sites.google.com/nyu.edu/nyu-hpc/hpc-systems/greene/hardware-specs) for list of GPUs):
``srun --cpus-per-task=12 --gres=gpu:v100:1 --mem=32GB --time=2:00:00 --account=pr_###_function --pty /bin/bash``  

Note that, although the specific GPU selection option is enabled, it is discouraged. Apparently the v100's are high in demand and it might take long (days?) waiting for the job to start. 

After the interactive job starts, your session will auto-login into a GPU node. Navigate to your project folder (under your `/home` or `/scratch`) and - 

```
$ ./singrw # Assuming you have already created a container image at the location, start it in read-write mode.
$ conda install jupyterlab
$ jupyter lab --no-browser -port=9000 -ip=0.0.0.0
```

Copy the node address from the logs (e.g. gr034.hpc.nyu.edu) then tunnel from your local machine - 

```
ssh -L 9000:gr034.hpc.nyu.edu:9000 <Net ID>@greene.hpc.nyu.edu
```

Access Jupyter Lab on your local machine by navigating your browser to https://localhost:9000

<br/>

### Running a batch job

Create a bash script (lets call it `sbatch_script.sh`) in your project directory with the following content (update the details relevant to your project):

```
#!/bin/bash -l

##############################
#       Job blueprint        #
##############################

# Give your job a name, so you can recognize it in the queue overview
#SBATCH --job-name=<Nice Job Name>

# Define, how many nodes you need. Here, we ask for 1 node.
# Each node has 16 or 20 CPU cores.
#SBATCH --cpus-per-task=12
#SBATCH --mem=32GB
#SBATCH --time=48:00:00
#SBATCH --account=pr_###_general

cd /scratch/<Net ID>/myproject/

./sing << EOF
conda init
conda activate <myproject conda env if needed>
python <myproject.py>

EOF
```

Submit this script as a batch job: 

```
sbatch sbatch_script.sh
```

To see the status of your batch job:

```
squeue -u <Net ID>
```

Alternatively, you can log on to https://ood-5.hpc.nyu.edu/pun/sys/dashboard to check the status of your job. 

<br/>

## Hugging Face Hacks

The huggingface_cache gets full too quickly (with downloaded models, sample datasets etc). When you install Hugging Face CLI, it sets up this cache directory under your `$HOME`. You will need to move this directory folder to `$HOME/scratch`. 

These instructions are for Mac users (Windows TODO).  
In your `~/.bashrc`:

``
export HF_HOME="/home/<Net ID>/scratch/huggingface_cache"
``

Then move your installed huggingface_cache directory to this newly created directory - 

``
mv ~/.cache/huggingface/ /home/<Net ID>/scratch/huggingface_cache/
``

Be sure to "source ~/.bashrc" before running HF commands after this. 


## Open On-Demand (OOD)

TODO


## Using VSCode to debug code using GPU Cluster

TODO


<!-- # Computing resources

While much of the day-to-day work we do in the lab can be done on a personal computer, our primary resource
for computing is the [High-Performance Computing (HPC) cluster](https://sites.google.com/nyu.edu/nyu-hpc).
The HPC can be used for large distributed jobs, but also provides a notebook server for a more interactive
workflow that you may be used to.

To use the HPC, you will need to apply for an account, and list your PI as a faculty sponsor.
See the [documentation](https://sites.google.com/nyu.edu/nyu-hpc/accessing-hpc/getting-and-renewing-an-account) for details on how to do this.

Especially if you are collaborating with other lab members, it will be helpful to become a member of the `marl` user group on HPC.
This is only possible after your HPC account has been created, and you will need to ask your PI to request this from the HPC administrators on your behalf.

# Recommendations for HPC use

First, make sure to read through the documentation on how to access the cluster.
If you are off-campus, you will need to go through either the VPN or an SSH gateway, as described [here](https://sites.google.com/nyu.edu/nyu-hpc/accessing-hpc#h.5v318r5hu99p).

You will need some basic familiarity with the UNIX command line, see [here](unix) for a quick overview.-->




