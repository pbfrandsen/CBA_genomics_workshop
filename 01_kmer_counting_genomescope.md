# 01: kmer counting with KMC and genome size estimation with GenomeScope2

In this portion of the workshop, we will take a fastq file containing PacBio HiFi reads and, first, use [`KMC`](https://github.com/refresh-bio/KMC) to count kmers and then use [`GenomeScope 2`](http://genomescope.org/genomescope2.0/) to estimate genome size, heterozygosity, and repeat content from the kmers.

The default raw data format from the PacBio revio is a `bam` file that contains the HiFi reads. Most of the downstream software works with `FASTQ` file, so we will use a `FASTQ` file that I have already made for you. You can find this at `/grphome/fslg_nanopore/nobackup/archive/genomics_workshop_byu_may_24/m54336U_230309_163624.hifi_reads.fastq.gz`.

Note: if you want to convert your `bam` file to a `fastq` file, there are multiple ways to do this, but I often use [`bedtools`](https://bedtools.readthedocs.io/en/latest/). A command that you can use to do this with bedtools is:

```
$ bedtools bamtofastq -i example.bam -fq example.fastq
```

Since these files contain a lot of data, I also usually like to compress them with `gzip`. You can do this simply with:

```
$ gzip example.fastq
```

`gzip` will automatically append a `.gz` to the filename.

However, for sake of saving time, we'll go ahead and get started with the raw reads from the `fastq` file that I've provided.


### kmer counting with `KMC`
First, navigate to your `~/nobackup/autodelete` directory. On the BYU supercomputer, this is where you should do all analysis. The material in the folders is periodically removed, so it is important to move important result to your home or archive directory. There is more limited space in your home directory so it is not well-suited for genomics analysis. It is easy to get to your autodelete directory from anywhere in the supercomputer. Simply type:

`$ cd ~/nobackup/autodelete`

Then you'll want to copy the `fastq.gz` file from the caddisfly, _Arctopsyche grandis_ into your directory. You can simply do this with:
```
$ cp /grphome/fslg_nanopore/nobackup/archive/genomics_workshop_byu_may_24/m54336U_230309_163624.hifi_reads.fastq.gz .
```

Remember to include the `.`. That is indicates that you want the file destination to be your current working directory. Now, make a new file called `files.txt` that contains the names of all of the files that you want to assess. In this case, we only have a single `FASTQ` file, but you could be using multiple sequencing runs and could add those all to your `files.txt` file. You can create this file with:

`$ ls *fastq.gz > files.txt`

Next, you'll need to create a job file. Remember, you can do this with the [job script generator](https://rc.byu.edu/documentation/slurm/script-generator). Choose a single node, 24 cores, and 6 hours. `KMC` can take a lot of RAM so select 14 GB of RAM per CPU. Then make a job file, perhaps called `kmc.job` and paste in the output from the job script generator. Once that it pasted in, make sure you add the commands to load the miniconda module and activate the `KMC` environment:

```
source /grphome/fslg_pws472/.bashrc
conda activate kmc
```

You'll also want to add a few job commands. First, make a `tmp` folder, which will hold some of the analysis files and then the kmc commands to count all 21mers and then generate a histogram file from those counts. Keep in mind that you set the `-k` parameter to what you want your kmer size to be and the `-t` parameter to the number of threads/CPUs you are using:

```
kmc -k21 -t24 -m300g -ci1 -cs10000 @files.txt reads tmp/
kmc_tools transform reads histogram reads.histo -cx10000
```

Then, you can go ahead and run this with:

```
$ sbatch kmc.job
```

This should complete in about 15 minutes, but may vary depending on how busy the cluster is. We can take a little break. If your job is still not running, then you can go ahead and copy the `reads.histo` file that I already made to take into `GenomeScope2`. It is in:

`/grphome/fslg_nanopore/nobackup/archive/genomics_workshop_byu_may_24/reads.histo`.

This is a good time to practice copying files over (with `cp`) and/or downloading files (I like to use `scp`).

Once your `reads.histo` file is downloaded to your computer, navigate to the [GenomeScope2 webserver](http://genomescope.org/genomescope2.0/). Once there, you can enter a description for your job, ensure that the kmer length is set to `21` and the ploidy to `2`. Obviously if you used a different kmer length to count kmers or have a polyploid organism, you would change those values.

Now, go ahead and drag and drop your `reads.histo` file into the box and scroll down and click on `Submit`. Now, your job will run and `Genomescope2` will fit a model to your kmer histogram. Once you are ready, let's explore the output together.
