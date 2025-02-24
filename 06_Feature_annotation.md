# Feature annotation with BRAKER3
###### Last updated: January 29, 2025
------------------------------------------------------------------------

### Prerequisites: 

- Soft-masked reference genome (you can use RepeatMasker and RepeatModeler, we used Earl Grey)
- RNASeq data in .bam format, mapped to the reference genome
- Protein data, preferably downloaded from [`OrthoDB`](https://bioinf.uni-greifswald.de/bioinf/partitioned_odb11/)

We already have the above files prepared. However, when running BRAKER3 on your own data, follow the below steps to prepare your reference genome, RNASeq data and protein data. BRAKER can also be run with only RNASeq data, or with only protein data. For more information on BRAKER, check their [`github`](https://github.com/Gaius-Augustus/BRAKER#f19)

We can skip over these steps directly to the "Running BRAKER3" below.

Here is the softmasked genome:

```bash
cp /grphome/fslg_nanopore/nobackup/archive/genomics_workshop_byu_may_24/arcto_4_HiC_chrom_EarlGrey/arcto_4_HiC_chrom_summaryFiles/arcto_4_HiC_chrom.softmasked.fasta .
```

Here is the protein database folder and RNASeq data:

```bash
cp /grphome/fslg_nanopore/nobackup/archive/genomics_workshop_byu_may_24/braker-files/* .
```

BRAKER doesn't work well with long fasta header names. Check your masked reference genome and confirm that there are no spaces, special symbols, or long names with grep:

```bash
grep ">" <filename>
```

The following code removes everything in fasta headers after a space:

```bash
 cut -d ' ' -f1 <masked_reference> > <outputfile>
```

BRAKER expects you to use [`hisat2`](https://daehwankimlab.github.io/hisat2/download/) to map RNASeq to your masked reference genome. Activate hisat2:

```bash
source /grphome/fslg_pws472/.bashrc
conda activate hisat2
```

Now index the reference genome with hisat2-build, and then map the index to the .fastq files.

```bash
# Index the reference genome:
hisat2-build arcto_4_HiC_chrom.softmasked.fasta arcto
# Map index to .fastq files:
hisat2 -x arcto -1 SRR2083574_1.fastq -2 SRR2083574_2.fastq -S arcto-alignment.bam
```

Convert .sam to .bam
```bash
conda deactivate
conda activate samtools # activate environment with samtools
samtools view -bS -o arcto-alignment.bam arcto_alignment.sam #convert .sam to .bam
samtools sort arcto-alignment.bam > arcto-sorted.bam #sort .bam
```

To run this in a job, it would look something like:

```
#!/bin/bash

#SBATCH --time=4:00:00   # walltime
#SBATCH --ntasks=1   # number of processor cores (i.e. tasks)
#SBATCH --nodes=1   # number of nodes
#SBATCH --mem-per-cpu=12240M   # memory per CPU core
#SBATCH -J "hisat"   # job name
#SBATCH --mail-user=<your_email@email.com>   # email address
#SBATCH --mail-type=BEGIN
#SBATCH --mail-type=END
#SBATCH --mail-type=FAIL


# Set the max number of threads to use for programs using OpenMP. Should be <= ppn. Does nothing if the program doesn't use OpenMP.
export OMP_NUM_THREADS=$SLURM_CPUS_ON_NODE

# LOAD MODULES, INSERT CODE, AND RUN YOUR PROGRAMS HERE
source /grphome/fslg_pws472/.bashrc
conda activate hisat2
hisat2-build arcto_4_HiC_chrom.softmasked.fasta arcto
hisat2 -x arcto -1 SRR2083574_1.fastq -2 SRR2083574_2.fastq -S arcto-alignment.sam
conda deactivate
conda activate samtools
samtools view -bS -o arcto-alignment.bam arcto-alignment.sam
samtools sort arcto-alignment.bam > arcto-sorted.bam
```

### Running BRAKER3
BRAKER has many dependencies, including AUGUSTUS, which also has many dependencies. This makes it a pain to install manually. Also, because of licensing issues, the conda environment for BRAKER is also difficult to configure (and not officially supported by the developers). Instead, the developers set up a singularity/apptainer to run BRAKER. This process takes several hours, so instead, we'll have you just copy the singularity image to your directory.

```
cp /nobackup/archive/grp/fslg_nanopore/genomics_workshop_byu_may_24/braker3.sif .
```

This will place the braker3.sif file into your directory. For future reference, if you have singularity/apptainer on your HPC and want to set up BRAKER, you can use the following commands (don't perform these for this workshop!):

```bash
module load spack
module load apptainer
apptainer build braker3.sif docker://teambraker/braker3:latest # this builds the .sif file which singularity/apptainer will need to run BRAKER
```

Once you have the `braker3.sif` file copied over, you can get the full path to it with:

```
realpath braker3.sif
```

Copy the path and set a new environmental variable with that path.

```
export BRAKER_SIF=<full_path_to.sif_file>
```

We need to download a config folder from AUGUSTUS to our own directory to make it writable for BRAKER. I already downloaded the config folder, you can copy it to your own directory: 

```bash
cp -r /grphome/fslg_nanopore/nobackup/archive/genomics_workshop_byu_may_24/config/ .
```

Now we'll copy three test files to confirm BRAKER is running correctly (make sure you are in your `~/nobackup/autodelete` directory):

```bash
cp /grphome/fslg_nanopore/nobackup/archive/genomics_workshop_byu_may_24/test*sh
```

This will copy three files, `test1.sh`, `test2.sh`, and `test3.sh`. Just for reference, I originally copied these from the singularity container as below.

```bash
apptainer exec --no-home -B $PWD:$PWD braker3.sif cp /opt/BRAKER/example/singularity-tests/test1.sh .
apptainer exec --no-home -B $PWD:$PWD braker3.sif cp /opt/BRAKER/example/singularity-tests/test2.sh .
apptainer exec --no-home -B $PWD:$PWD braker3.sif cp /opt/BRAKER/example/singularity-tests/test3.sh .
```

I then edited them to add the value `--no-home` directive, commented out a section that deleted existing output directories, and set a value for the `AUGUSTUS_CONFIG_PATH`. `test1.sh`, for example now has executable code that looks like:

```
singularity exec --no-home -B ${PWD}:${PWD} ${BRAKER_SIF} braker.pl --AUGUSTUS_CONFIG_PATH=${PWD}/config --genome=/opt/BRAKER/example/genome.fa --bam=/opt/BRAKER/example/RNAseq.bam --workingdir=${wd} --GENEMARK_PATH=${ETP}/gmes --threads 8 --gm_max_intergenic 10000 --skipOptimize --busco_lineage eukaryota_odb10 &> test1.log
```

Before we run these test files, we have to manually place the eukaryota_odb10 OrthoDB files into new `test1`, `test2`, and `test3` directories. This is because, otherwise, `braker3` will automatically attempt to download them from OrthoDB and the software will throw an error since the compute nodes are not connected to the internet. You can do that with the following:

```
cp -r /grphome/fslg_nanopore/nobackup/archive/genomics_workshop_byu_may_24/mb_downloads .
mkdir test1
mkdir test2
mkdir test3
cp -r mb_downloads test1
cp -r mb_downloads test2
cp -r mb_downloads test3
```

We can then test whether the setup worked by running a job that runs the test files. Navigate to the [Job Script Generator](https://rc.byu.edu/documentation/slurm/script-generator) and create a new job script with 8 processor cores, 10 GB of RAM and 2 hour wall time. Copy and paste the scrip into a new job file called `braker3_test.job`. Add the following to the bottom of your job file:

```bash
module load spack
module load apptainer
export BRAKER_SIF=${PWD}/braker3.sif
bash test1.sh
bash test2.sh
bash test3.sh
```

Now, go ahead and inspect the output. You can view the logs and also look in the directories for each test file.

If that worked, we are now preapred to run BRAKER3 on our data! We won't complete the steps below because the job takes 1-3 days. However, I will keep these steps available in the case you want to return and run these steps on your own data.

First, make a new directory and copy over the orthoDB gene set:

```
mkdir arcto_grandis_annot
cp -r mb_downloads arcto_grandis_annot
```

Create a new job script like the one below to run BRAKER. 


```bash
nano braker.job
```

```bash
#!/bin/bash

#SBATCH --time=72:00:00   # walltime
#SBATCH --nodes=1   # number of nodes
#SBATCH --mem-per-cpu=8192M   # memory per CPU core
#SBATCH --ntasks=12   # number of processor cores (i.e. tasks)
#SBATCH --mail-user=<youremail@email.com>   # email address
#SBATCH --mail-type=BEGIN
#SBATCH --mail-type=END
#SBATCH --mail-type=FAIL
#SBATCH --job-name= <name>

# Load modules
module load spack
module load apptainer

# Get the SIF file into the path
export BRAKER_SIF=${PWD}/braker3.sif
export AUGUSTUS_CONFIG_PATH=${PWD}/config
# Run singularity/apptainer
apptainer exec --no-home -B $PWD:$PWD braker3.sif braker.pl \
--genome=arcto_4_HiC_chrom.softmasked.fasta \
--bam=arcto-sorted.bam \
--prot_seq=Arthropoda.fa \
--workingdir=arcto_grandis_annot \
--gff3 \
--threads=$SLURM_NTASKS \
--species=Arctopsyche-grandis \
--AUGUSTUS_CONFIG_PATH=${PWD}/config \
--busco_lineage insecta_odb10
```

It's important to understand what each of the above lines is doing. 
`--genome` references the softmasked reference genome 
`--bam` references the .bam RNASeq file mapped to your genome
`--prot_seq` is the protein database downloaded from OrthoDB
`--workingdir` specifies the working directory all files will be saved into - change this each time you run BRAKER so data isn't overwritten
`--gff3` requests output to be in gff3 format
`--species` is a unique species identifier
`--AUGUSTUS_CONFIG_PATH` is the file path to the config directory that we copied above

### Examining the output:

Once the job is finished, you can take a quick look at some stats to see how the different gene annotations performed within `BRAKER3` compare. To do this you can use a script from GALBA:

```bash
wget https://raw.githubusercontent.com/Gaius-Augustus/GALBA/master/scripts/analyze_exons.py
chmod u+x analyze_exons.py
./analyze_exons.py -f arcto_grandis_annot/braker.gtf
./analyze_exons.py -f arcto_grandis_annot/Augustus/augustus.hints.gtf
./analyze_exons.py -f arcto_grandis_annot/GeneMark-ETP/genemark.gtf
```
