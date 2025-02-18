# Identifying contaminants with BlobTools

[BlobTools](https://github.com/DRL/blobtools) combines the read-depth, GC content, and closest taxonomic match of each contig to identify likely contaminants. Full documentation can be found [here](https://blobtools.readme.io/docs/what-is-blobtools).

Note: The Blast and Minimap2 steps can run independently of each other, so go ahead and get both of those running. Using `BLAST` to search all of the contigs may take anywhere from several hours to a few days depending on the size of the genome. Feel free to look through the rest of the steps and explore some of the example ouputs found in `/grphome/fslg_nanopore/nobackup/archive/genomics_workshop_byu_may_24/blobtools`

First, let's make a directory for our output:

```
mkdir blobtools
cd blobtools
```

## Blast
Create a job script to blast each contig of the genome against the nucleotide (nt) database:
```
nano blast_contigs.job
```
Copy the script below into your the file. Make sure to edit it to include your own email and file paths.

```
#!/bin/bash

#SBATCH --time=72:00:00   # walltime
#SBATCH --ntasks=30   # number of processor cores (i.e. tasks)
#SBATCH --nodes=1   # number of nodes
#SBATCH --mem-per-cpu=21240M   # memory per CPU core
#SBATCH -J "BlobTools blast"   # job name
#SBATCH --mail-user=<youremail@email.com>   # email address
#SBATCH --mail-type=BEGIN
#SBATCH --mail-type=END
#SBATCH --mail-type=FAIL


# Set the max number of threads to use for programs using OpenMP. Should be <= ppn. Does nothing if the program doesn't use OpenMP.
export OMP_NUM_THREADS=$SLURM_CPUS_ON_NODE

# LOAD MODULES, INSERT CODE, AND RUN YOUR PROGRAMS HERE
source /grphome/fslg_pws472/.bashrc
conda activate blast

blastn \
-task megablast \
-db /apps/blast/databases/nt \
-query <path_to_contigs> \
-outfmt "6 qseqid staxids bitscore std" \
-max_target_seqs 20 \
-max_hsps 1 \
-evalue 1e-20 \
-num_threads $SLURM_NTASKS \
-out <genome_name>.blast.out
```


Explantion of inputs:

```
-db: ncbi nucleotide database
-query: input file (FASTA)
-outfmt: format of the output file (important to for blobtools) 
-max_target_seqs: Number of aligned sequences to keep.
-max_hsps: Maximum number of HSPs (alignments) to keep for any single query-subject pair.
-num_threads: number of CPUs
-out: name of the output file
```
Start the job by running:

```
sbatch blast_contigs.job
```

This job can take a few days to run.

## Minimap2
BlobTools will use the alignment data from Minimap2 to calculate the sequencing coverage for each contig. 
You can download the precompiled binary for Minimap2 into your blobtools directory by running the following:

```
curl -L https://github.com/lh3/minimap2/releases/download/v2.28/minimap2-2.28_x64-linux.tar.bz2 | tar -jxvf -
```

Next, create a job script to map the reads to the genome:
```
nano map_contigs.job
```

Copy the script below into the file. Make sure to edit it to include your own email and file paths.

```
#!/bin/bash

#SBATCH --time=15:00:00   # walltime
#SBATCH --ntasks=20   # number of processor cores (i.e. tasks)
#SBATCH --nodes=1   # number of nodes
#SBATCH --mem-per-cpu=20144M   # memory per CPU core
#SBATCH -J "BlobTools minimap2"   # job name
#SBATCH --mail-user=<youremail@email.com>   # email address
#SBATCH --mail-type=BEGIN
#SBATCH --mail-type=END
#SBATCH --mail-type=FAIL


# Set the max number of threads to use for programs using OpenMP. Should be <= ppn. Does nothing if the program doesn't use OpenMP.
export OMP_NUM_THREADS=$SLURM_CPUS_ON_NODE

# LOAD MODULES, INSERT CODE, AND RUN YOUR PROGRAMS HERE


module load samtools

./minimap2-2.28_x64-linux/minimap2 \
-ax map-hifi \
-t $SLURM_NTASKS \
<path_to_contigs> \
<path_to_raw_sequencing_reads> \
| samtools sort -@$SLURM_NTASKS -O BAM -o <genome_name>_sorted.bam

samtools index <genome_name>_sorted.bam
```

Explanation of inputs:

```
minimap2
-ax: preset configuration to map hifi reads to genomes.
-t: number of threads to use.
samtools
sort: sort command
-@: number of threads to use.
-O: output format.
-o: name of the outputformat
index: index command
```

You can then submit the job with:

```
sbatch map_contigs.job
```

## BlobTools
### Creating the database
Now we will combine our previous outputs to create a BlobTools database. 

To do this, create a job script to run BlobTools:

```
nano blobDB.job
```

Copy the script below into the file. Make sure to edit it to include your own email and file paths.

```
#!/bin/bash

#SBATCH --time=24:00:00   # walltime
#SBATCH --ntasks=1   # number of processor cores (i.e. tasks)
#SBATCH --nodes=1   # number of nodes
#SBATCH --mem-per-cpu=12288M   # memory per CPU core
#SBATCH -J "blobplots"   # job name
#SBATCH --mail-user=<youremail@email.com>   # email address
#SBATCH --mail-type=BEGIN
#SBATCH --mail-type=END
#SBATCH --mail-type=FAIL


# Set the max number of threads to use for programs using OpenMP. Should be <= ppn. Does nothing if the program doesn't use OpenMP.
export OMP_NUM_THREADS=$SLURM_CPUS_ON_NODE

# LOAD MODULES, INSERT CODE, AND RUN YOUR PROGRAMS HERE

source /grphome/fslg_pws472/.bashrc
conda activate blobtools

blobtools create -i <path_to_genome_fasta> \
-b <path to minimap2 output in BAM format> \
-t <path to blast output> \
-o <genome name>_blobplot
```
Explanation of inputs:

```
-i: genome assembly (fasta)
-b: mapped reads to genome assembly (bam)
-t: hits output file from a search algorith (i.e blastn). hit file is a TSV file which links sequence IDs in a assembly to NCBI TaxIDs, with a given score.
-o: path and/or name of the blobtools database.
```

You can submit this job with:

```
sbatch blobDB.job
```

### Making the plots

```
source /grphome/fslg_pws472/.bashrc
conda activate blobtools
mkdir plots
blobtools plot -i <genome name>_blobplot.blobDB.json -o plots/
```

To view the PNG outputs, you will need to download them to your own computer. To do this, open up command prompt on your own machine and use the following command:

```
scp [USERID]@ssh.rc.byu.edu:<path to blobtools folder>/plots/* <path to where you want to save these on your computer>
```

You will then need to enter you password and a verification code for the files to copy over. Open up the PNGs to view possible contaminant contigs.


### Filtering out contaminant contigs
Create a table view of the json database. You can choose the taxonomic level that will work best for filtering out contaminants. The availale options are listed in the explanation of the inputs.

```
blobtools view -i <blobDB in json format> -r <desired taxonomic level>
```

Explanation of inputs:

```
-i, --input <BLOBDB>        BlobDB file (created with "blobtools create")
-r, --rank <TAXRANK>...     Taxonomic rank(s) at which output will be written.
                                    (supported: 'species', 'genus', 'family', 'order',
                                    'phylum', 'superkingdom', 'all') [default: phylum]
```

(Additional options can be seen at https://blobtools.readme.io/docs/view)

We can now filter through the table to identify which contigs are likely contaminants. Choose the taxon your organism *does* belong to (at whatever taxonomic level you chose when creating the table).
````bash
grep -v <DESIRED_TAXON> <blobDB ending in .table.txt>
````
The -v option inverts the grep results so that rows *with* the taxon are *excluded*. This leaves you with a list of contanimant contigs. To isolate just the contig names, do the following:

```
grep -v <DESIRED_TAXON> <blobDB ending in .table.txt> | awk '{print $1}' > contaminant_contigs.txt
```

BlobTools has a built-in module called **seqfilter** to filter your genome based on a list like this.

```
blobtools seqfilter -i <genome_name>.fasta -l contaminant_contigs.txt -v
```

Explanation of inputs:

```
-i, --infile <FASTA>        FASTA file of sequences (Headers are split at whitespaces)
-l, --list <LIST>           TXT file containing headers of sequences to keep (or to discard since we will use the -v option)
-v, --invert                Invert filtering (Sequences w/ headers NOT in list)
```

You can also pursue other options for filtering out contaminants, such as setting thresholds for GC content or sequencing coverage and the removing contigs that fall oustide of your parameters.