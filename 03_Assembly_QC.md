# 03: Assembly QC

Now that we have a whole genome assembly, we're going to do some quick checks to see how it looks. The first thing that we will do is estimate the contig statistics with [`QUAST`](https://github.com/ablab/quast). Then we will run `compleasm` to get a handle on genome completeness.

### QUAST

First, make sure that your assembly file was written appropriately from `hifiasm`. In the last lab, we converted three files from `gfa` format to `fasta` format. The one that we will use from here on is the one that ends in `p_ctg.fasta`. This is the `primary` assembly. Perhaps sometime soon, we'll have more tools that work directly on genome graphs that include the haplotypic diversity in the genome, but most tools currently work with a single fasta file representation of the genome.

Next, we'll go ahead and run `QUAST` on that assembly file. First, load the `conda` module and then activate the `QUAST` environment.

```
source /grphome/fslg_pws472/.bashrc
conda activate quast
```

Then we can simply run quast on the assembly file. It is pretty fast so we can run it interactively and won't need to put it into a job file.

```
quast <assembly_name>.bp.p_ctg.fasta
```

Now, `QUAST`, will evaluate your genome assembly and the results will be added to `quast_results/latest`. Once your run is complete, you can navigate to that folder. There should be a file called `report.pdf`. Go ahead and download that file with `scp` and examine it's contents.

### compleasm

[`compleasm`](https://github.com/huangnengCSU/compleasm) was designed to evaluate your genome assembly for the presence of universal single copy orthologs. These gene sets are generated from [`OrthoDB`](https://www.orthodb.org/).

Create a new job script using the [Job Script Generator](https://rc.byu.edu/documentation/slurm/script-generator). Make sure that the job is limited to one node, select 4 processor cores, 4 GB of RAM per processor, and a wall-time of 12 hours. Copy the resulting script into a new job file called `compleasm.job`.

Download the appropriate lineage to your working folder (the one containing your genome assembly file). It should download the lineage into a folder called `mb_downloads`. You can see a list of lineages [here](https://busco-data.ezlab.org/v5/data/lineages/). Replace "insecta" with the appropriate lineage in the code below.
```
source  /grphome/fslg_pws472/.bashrc
conda activate compleasm
compleasm download insecta
```
Open the job file and add the following commands to the bottom, separated by line breaks:

```
source  /grphome/fslg_pws472/.bashrc
conda activate compleasm
compleasm run -a <path to assembly> -o compleasm -t $SLURM_NTASKS -l insecta -L mb_downloads -m busco
```

Note, that you should substitute your particular genome assembly file for "arctopsyche.p_ctg.fasta". Note, we are using the `insecta_odb10` dataset. For some taxa, there are more specific datasets that you could use. For example, there is a `lepidoptera_odb10` dataset and for holometabolous insects, there is a `endoptergyota_odb10` dataset.

### tidk

Another way to evaluate the quality of your genome assembly is to see how many contigs contain telomeres. With high-quality, long reads, we can often assemble whole chromosomes! You can verify this by the presence of telomere sequence at both ends of the contig. There is a nice tool developed by the Darwin Tree of Life team to do this called [`tidk`](https://github.com/tolkit/telomeric-identifier).

Now, go ahead and change directories into the directory containing your assembled genome. Now load the conda module and activate the conda environment with tidk installed.

```
source /grphome/fslg_pws472/.bashrc
conda activate tidk
```

Next, we're going to search for telomere sequences, using clade-specific telomere sequences. If you are using the sample data, this will be `Trichoptera`. However, I know some of you are analyzing other genomes and you might be able to find the clade in the documentation for [`tidk`](https://github.com/tolkit/telomeric-identifier). I know there are clade specific sets for `Lepidoptera`, `Hymenoptera`, and `Plecoptera`. If your clade isn't listed, you can try the closest related clade, or, alternatively, try to find the telomere sequence from an earlier publication and use `tidk search` instead of `tidk find`.

`tidk` is pretty fast so you can just run it interactively. Here is an example command:

```
tidk find -c Trichoptera -o arcto_tidk -d tidk_results arctopsyche.asm.p_ctg.fasta
```

After the finding it finished, you can then make plots with:

```
tidk plot -t tidk_results/arcto_tidk_telomeric_repeat_windows.tsv
```

Now, you'll have a new image file called `tidk-plot.svg` that you can download to check out whether your contigs have telomeres!
