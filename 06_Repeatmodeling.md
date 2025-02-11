# Repeat modeling and Repeat masking
###### Last updated: January 29, 2025
------------------------------------------------------------------------

Earl Grey is a transposable element annotation pipeline. It combines both `RepeatModeler` and `RepeatMasker`, and provides almost all the output files you could possibly want. 

It's also pretty simple to run! For the full documentation see the [`website`](https://github.com/TobyBaril/EarlGrey?tab=readme-ov-file#recommended-installation-with-conda-or-mamba)

First, we'll install and activate the environment:

```bash
source /grphome/fslg_pws472/.bashrc
conda activate earlgrey
```

Let's create a directory to save our output to:
```bash
mkdir earl-out
```

Now we'll create a job script to run Earl Grey:

```bash
nano earlgrey.job
```

Copy and paste the job script below into your job file, editing it to match your file names, species and email.

```bash

#!/bin/bash

#SBATCH --time=72:00:00   # walltime
#SBATCH --ntasks=24   # number of processor cores (i.e. tasks)
#SBATCH --nodes=1   # number of nodes
#SBATCH --mem-per-cpu=24G   # memory per CPU core
#SBATCH -J "earl"   # job name
#SBATCH --mail-user=<youremail@email.com>   # email address
#SBATCH --mail-type=BEGIN
#SBATCH --mail-type=END
#SBATCH --mail-type=FAIL

# Set the max number of threads to use for programs using OpenMP. Should be <= ppn. Does nothing if the program doesn't use OpenMP.
export OMP_NUM_THREADS=$SLURM_CPUS_ON_NODE

# activate environment with Earl Grey
source /grphome/fslg_pws472/.bashrc
conda activate earlgrey

#run Earl Grey
earlGrey -g <genome_name> -t $SLURM_NTASKS -s <species_name> -r arthropoda -d yes -o ../earl-out/
```

Start the job!

```bash
sbatch earlgrey.job
```

There are a lot of optional parameters to include in Earl Grey, we're only including a few of them:
- `-t == number of threads`
- `-r == search term for RepeatMasker`
- `-d == create a soft-masked genome (we will need this later in the week, default is no)`

Earl Grey usually runs for 1-3 days, depending on the size and complexity of the genome. While your job is running we're going to look at output files from a completed run. Copy over the Earl Grey output files from the shared directory:

```bash
cp -r /home/youruserid/fsl_groups/fslg_nanopore/nobackup/archive/genomics_workshop_byu_may_24/ .
```

Move into the `*summaryFiles` directory using `cd`. You'll see there are several files there, including two pdfs. Download the .pdf files to your computer. Which transposable element is found in highest frequency? Have there been any recent shifts in transposable element activity?

Check out the other files in the `*summaryFiles` folder using head. The `.gff` and `.bed` file provide coordinates and the contig name for each of the transposable elements. `*combined_library.fasta` has all of the transposable elements found in the genome. 

Copy the .bed file into a new directory, we'll need it for the intersect analysis.

```bash
cp *.bed ~/nobackup/autodelete/
```

The hard-masked and soft-masked genomes are in a different folder. Move into `*RepeatMasker`. View the first few lines of each file using `head <filename>`. Which file is hard-masked? Which is soft-masked? 

Now we'll look at the intersect between heavy-chain fibroin and our estimated transposable elements. For this we need a `.gff` file with coordinates for h-fibroin. You'll find that in the shared folder, copy it into your personal directory:

```bash
cd ~/nobackup/autodelete/
cp /home/youruserid/fsl_groups/fslg_pws472/nobackup/archive/spin-workshop/Arcto-hfib.gff .
```

Deactivate the `earlgrey` environment and activate bedops to convert `Arcto-hfib.gff` to `.bed`:

```bash
conda deactivate
conda activate bedops
gff2bed < Arcto-hfib.gff > Arcto-hfib.bed
```

Deactivate the `bedops` environment and activate `bedtools` to sort both bedfiles:

```bash
conda deactivate
conda activate bedtools
bedtools sort -i arcto_4_HiC_chrom.filteredRepeats.bed > ArcGran_repeats_sorted.bed
bedtools sort -i Arcto-hfib.bed > Arcto-hfib-sorted.bed
```

If you get an error that the start is greater than the end run the following code to remove those lines:

```bash
awk '{OFS="\t"; if ($5-$4 > 0) print}' arcto_4_HiC_chrom.filteredRepeats.bed > arcto_4_filteredRepeats-presort.bed
```

If you want to additionally look at the flanking region, run the code below to pull flanks:

```bash
conda deactivate
conda activate samtools
samtools faidx genome.fna
conda deactivate
conda activate bedtools
bedtools flank -i Arcto-hfib-sorted.bed -g path/to/genome/file.fai -b <flanking bp> > Arcto-hfib-flank.bed
```

Now we can run the intersect analysis!

```bash
bedtools intersect -a ArcGran_repeats_sorted.bed -b Arcto-hfib-sorted.bed > ArcGran_intersect.bed
```

Now you can download the intersect file and the contig the intersect is found on and open them in your preferred sequence viewer!

