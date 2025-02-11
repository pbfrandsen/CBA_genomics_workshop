# 02: Whole Genome Assembly

[Hifiasm](https://github.com/chhylp123/hifiasm) is a tool for assembling genomes, especially for PacBio HiFi reads. It is available in `bioconda`, but also as source code that you can download and build if you so desire. There are a few different options that you can explore at the [hifiasm GitHub](https://github.com/chhylp123/hifiasm) if you feel so inclined. It is always good to become familiar with the various options.

We'll run `hifiasm` as a batch job (i.e., a task submitted to the supercomputer).  

We'll first create a job script, a file with parameters and commands to run our task. 

You can create a job script by using BYU's [Job Script Generator](https://rc.byu.edu/documentation/slurm/script-generator). 


Fill out the following parameters and options for your job script:

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; *Limit this job to one node: [select this option]*

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; *Number of processor cores across all nodes: 24* 

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; *Memory per processor: 14 GB* 

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; *Walltime: 24 hours* 

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; *Job name: [add your job’s name]*

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; *Receive email for job events: [click on begin, end, abort]*

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; *Email address: [add your email address]*


Then, scroll down and click on "Copy Script to Clipboard."

Go back to your terminal window and navigate to your `~/nobackup/autodelete` directory that holds the reads that you copied over earlier.

Use your preferred text editor (vim, vi, nano) to create a text file and name it `hifiasm.job`. Remember that file endings don't usually mean anything in `Unix` so you can name it whatever you want. I prefer `.job` to indicate that it is a job that I am submitting to the cluster. Many others use `.sh` or some other variant. Do whatever works for you, but I'd encourage you to be consistent. 

```bash
nano hifiasm.job 
```

Paste your job script in your newly created text file.  

Scroll to the bottom of your text file, add a few lines of space, and then include the lines below to activate hifiasm and run it along with some running options. 

```bash
source /grphome/fslg_pws472/.bashrc
conda activate hifiasm

hifiasm -o $1.asm -t $SLURM_NTASKS $2

awk '$1 ~/S/ {print ">"$2"\n"$3}' $1.asm.bp.p_ctg.gfa > $1.asm.bp.p_ctg.fasta
awk '$1 ~/S/ {print ">"$2"\n"$3}' $1.asm.bp.hap1.p_ctg.gfa > $1.asm.bp.hap1.p_ctg.fasta
awk '$1 ~/S/ {print ">"$2"\n"$3}' $1.asm.bp.hap2.p_ctg.gfa > $1.asm.bp.hap2.p_ctg.fasta
```

 The '-t' determines the number of CPUs or processor cores, and the '-o' indicates the output file prefix. 

The values that begin with `$` are argument or input you'll provide when submitting your job. In this case, we'll use `$1` to indicate the prefix of what you want to call the assembly and `$2` to indicate the read `FASTQ` file. 

The last three lines are `awk` commands to convert the default output of `hifiasm` (a genome graph format) to `fasta`, which is useful for downstream applications. 

 Your job script should look something like this: 

```bash
#!/bin/bash

#SBATCH --time=24:00:00   # walltime
#SBATCH --ntasks=24   # number of processor cores (i.e. tasks)
#SBATCH --nodes=1   # number of nodes
#SBATCH --mem-per-cpu=14096M   # memory per CPU core
#SBATCH -J "genome_assembly "   # job name
#SBATCH --mail-user=<your_email>   # email address
#SBATCH --mail-type=BEGIN
#SBATCH --mail-type=END
#SBATCH --mail-type=FAIL


# Set the max number of threads to use for programs using OpenMP. Should be <= ppn. Does nothing if the program doesn't use OpenMP.
export OMP_NUM_THREADS=$SLURM_CPUS_ON_NODE

# LOAD MODULES, INSERT CODE, AND RUN YOUR PROGRAMS HERE

source /grphome/fslg_pws472/.bashrc
conda activate hifiasm

hifiasm -o $1.asm -t $SLURM_NTASKS $2

awk '$1 ~/S/ {print ">"$2"\n"$3}' $1.asm.bp.p_ctg.gfa > $1.asm.bp.p_ctg.fasta
awk '$1 ~/S/ {print ">"$2"\n"$3}' $1.asm.bp.hap1.p_ctg.gfa > $1.asm.bp.hap1.p_ctg.fasta
awk '$1 ~/S/ {print ">"$2"\n"$3}' $1.asm.bp.hap2.p_ctg.gfa > $1.asm.bp.hap2.p_ctg.fasta
```
  

Save the changes you made and exit your text file window.  

Now, you can run `hifiasm` to assemble your organism's genome **Note** if you are using your own reads, you'll need to substitute the `fastq` file name with whatever filename contains your reads:  

```bash
sbatch hifiasm.job arctopsyche m54336U_230309_163624.hifi_reads.fastq.gz
```
Hit enter and you're done!

To check if your job is running and the state of your job, run the following command:

```bash
squeue -u $USER
```

The job should take an hour or two to finish and we will take a look at the finished product tomorrow! 
