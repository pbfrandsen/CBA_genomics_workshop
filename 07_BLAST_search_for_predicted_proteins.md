#BLAST search for predicted proteins

Now that you have a set of predicted proteins, there should be an output from `BRAKER3` called `braker.aa`. This contains the predicted peptides from you feature annotation. If you wanted to assign functional annotations to these proteins with, e.g. `BLAST2GO`, then you will first need to search the proteins with `BLAST`.

Copy that file to your directory. You can either use your own file or the one with predicted proteins from `BRAKER`. Go ahead and make a new folder, maybe called `protein_blast` and copy the file from this path:

```
/grphome/fslg_nanopore/nobackup/archive/genomics_workshop_byu_may_24/arctopsyche-braker.aa
```

To run `BLAST` across all 12,000 predicted proteins in a timely fashion, we will first split up the protein file into many files that we will run with a supercomputer array. First make a new directory called `fa` to contain the new fasta files.

```
mkdir fa
```

Here is a little snippet of code that will allow you to split up the file, taken from this helpful little post [https://www.biostars.org/p/13270/](https://www.biostars.org/p/13270/).

```
awk 'BEGIN {n_seq=0;} /^>/ {if(n_seq%50==0){file=sprintf("fa/arctopsyche_aa_%d.fa",n_seq);} print >> file; n_seq++; next;} { print >> file; }' < arctopsyche-braker.aa
```

Now change directories into that directory:

```
cd fa
```

Make a new directory called `xml`.

```
mkdir xml
```

Make a list of the sequences called `sequence_list.txt`

```
ls *fa > sequence_list.txt
```

Check how many sequences there are in your sequence list

```
wc -l sequence_list.txt
```

For _Arctopsyche_ it should be 246. Take note of your number if it is different so that you can put it into the job script.

Now, we're going to make a new job script that will run a `SLURM` array. This is a way to run the same job across multiple files. Here is an example of the job file for _Arctopsyche_ that I pasted into `arcto_blast.job`.

```
#!/bin/bash

#SBATCH --time=124:00:00   # walltime
#SBATCH --ntasks=1   # number of processor cores (i.e. tasks)
#SBATCH --nodes=1   # number of nodes
#SBATCH --mem-per-cpu=6144M   # memory per CPU core
#SBATCH -J "arcto_blast"   # job name
#SBATCH --array=1-246


# Set the max number of threads to use for programs using OpenMP. Should be <= ppn. Does nothing if the program doesn't use OpenMP.
export OMP_NUM_THREADS=$SLURM_CPUS_ON_NODE

# LOAD MODULES, INSERT CODE, AND RUN YOUR PROGRAMS HERE
source /grphome/fslg_pws472/.bashrc
conda activate blast
i=$SLURM_ARRAY_TASK_ID
P=`awk "NR==$i" sequence_list.txt`
blastp -query ${P} -db /apps/blast/databases/nr -outfmt 5 -max_target_seqs 10 -evalue 1e-4 -out xml/${P}.xml
```

It will output all of the `BLAST` results into xml files in the `xml` folder. You can then use these downstream for input into `BLAST2GO`.

Go ahead and start the array with:

```
sbatch arcto_blast.job
```