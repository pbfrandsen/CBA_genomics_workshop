# Gene Specific Annotation

## An example with heavy chain fibroin in *Arctopsyche grandis*

### Create a fasta file with the termini of a published h-fib

Use your preferred text editor (vim, vi, nano) to create a text file and name it `termini.fasta`.
```
nano termini.fasta
```

Paste the termini in fasta format. Here is an example of the format, feel free to copy and paste it into your fasta file.
```
>arcto_1_1_n_terminus
MRAAILLILFCSLQILTGATAHGDNVIGKLSDFLSHGHLGTNCGRHERILQGDDVIETNAKGEIIEKITSRKEILTDDD
>arcto_1_1_c_terminus
YSNVGGAHVPRGSNLYTTHPDPSTVLKSCKTSPYQLLIKVGNARKRNGNC
>arcto_1_2_n_terminus
MRAAILLILFCSLQILTGATAHGDNVIGKLSDFLSHGHLGTNCGRHERILQGDDVIETNAKGEIIEKITSRKEILTDDD
>arcto_1_2_c_terminus
YSNVGGAHVPRGSNLYTTHPDPSTVLKSCKTSPYQLLIKVGNARKRNGNC
```

### Make a blast database for your genome

Install BLAST or use the following module and conda environment (available to all with access to the `fslg_pws472` folder).
```
source /grphome/fslg_pws472/.bashrc
conda activate blast
```
Create a blast database for your genome with `makeblastdb`. You can just run this interactively rather than in a job script, it should only take a few seconds. 
Here's an example of what that might look like:
```
makeblastdb -dbtype nucl -in arcto_2_1.fasta
```
The `-dbtype nucl` indicates that it's a nucleotide database. After running this command you should see extra files in your directory (.ndb, .nin, .nhr, etc.)

### Run tblastn with the termini and genome

Use your preferred text editor (vim, vi, nano) to create a text file and name it `blast.sh`
```
nano blast.sh
```
Copy and paste the job script below into your job file, editing it to include your email.
```
#!/bin/bash

#SBATCH --time=24:00:00   # walltime
#SBATCH --ntasks=1   # number of processor cores (i.e. tasks)
#SBATCH --nodes=1   # number of nodes
#SBATCH --mem-per-cpu=4G   # memory per CPU core
#SBATCH -J "blast"   # job name
#SBATCH --mail-user=ashlynpowell913@gmail.com   # email address
#SBATCH --mail-type=BEGIN
#SBATCH --mail-type=END
#SBATCH --mail-type=FAIL
# LOAD MODULES, INSERT CODE, AND RUN YOUR PROGRAMS HERE

source /grphome/fslg_pws472/.bashrc
conda activate blast

species=$1 # from command line
genome="${species}.fasta"
output="${species}.out"
termini="termini.fasta"

tblastn -db $genome -query $termini -out $output -outfmt 7
```
Now submit the job script with `sbatch`. Make sure to give it the name of your genome when you run it (without the `.fasta`). For example:
```
sbatch blast.sh arcto_2_1
```
You can check that it's running with:
```
squeue -u <your_username>
```
When it's done you should have an output file called `<genome_name>.out` (ex: `arcto_2_1.out`) with the blast results in a table format. 

### Pull out region of the genome where h-fib is located

Install pandas and pyfaidx or use the following module and conda environment (available to all with access to the `fslg_pws472` folder).
```
source /grphome/fslg_pws472/.bashrc
conda activate pyfaidx
```
The following python script extracts the h-fib gene and flanking regions to a fasta file using the blast results. Paste it into a file named `gene_ext.py`.
```
#!/usr/bin/env python3
import sys
import os
import pandas as pd
from pyfaidx import Fasta

genome_name = sys.argv[1] # from the command line

b_result = genome_name + ".out"
genome = Fasta(genome_name + ".fasta")
flank=40000
print_messages = False

def check_flank(caps):
	if caps.get('first') < -1:
		caps['first'] = 0
	contig_length = len(genome[caps.get('contig')])
	if caps.get('last') > contig_length:
		caps['last'] = contig_length
	if print_messages: print(caps) 
	num_nucleotides = caps['first'] - caps['last']
	if print_messages: print(num_nucleotides)

def write_file(caps):
	if caps.get('correct_ori') == True:
		if print_messages: print("N to C = Negative")
		output = open(genome_name + "_gene.fa", "a")
		gene = genome[caps.get('contig')][int(caps.get('first')):int(caps.get('last'))]
		output.write(">" + genome_name + "_heavy_chain_fibroin(contig: " + str(contig) + ")\n" + str(gene) + "\n")
		output.close()
		print("Gene extracted")
	elif caps.get('correct_ori') == False:
		if print_messages: print("C to N = positive")
		output = open(genome_name + "_gene.fa", "a")
		gene = reverse_complement(genome[caps.get('contig')][int(caps.get('first')):int(caps.get('last'))])
		output.write(">"+ genome_name + "_heavy_chain_fibroin(contig: " + str(contig) + ")\n" + str(gene) + "\n")
		output.close()
		print("Gene extracted")
	
def gene_ext(b_reult, termini, flank):	#The df is made out of the output file of the blast run.
	df = pd.read_csv(b_result, sep="\t", header=None, names=["query acc.ver", "subject acc.ver", "% identity", "alignment length", "mismatches", "gap opens", "q. start", "q. end", "s. start", "s. end", "evalue", "bit score"])
	df = df.loc[(df["query acc.ver"]== termini + "_c_terminus") | (df["query acc.ver"]== termini + "_n_terminus")]
	if print_messages: print(df)
	
	if ((len(df) < 2) or df.empty):
		if print_messages: print("There wasn't enough to match the C and N termini. Change your termini sequences.")
		return None
	df = df.sort_values(by=["subject acc.ver", "% identity"], ascending=(False, False))
	df = df.loc[df['subject acc.ver'].duplicated(keep = False)]
	
	if ((len(df) < 2) or df.empty):
		if print_messages: print(df)
		if print_messages: print("There wasn't enough to match the C and N termini. Change your termini sequences.")
		return None
	df = df.sort_values(by=["% identity", "subject acc.ver"], ascending=(False, False))	
	df = df.drop_duplicates(subset = ["query acc.ver"], keep='first')
	
	if ((len(df) < 2) or df.empty):
		if print_messages: print(df)
		if print_messages: print("There wasn't enough to match the C and N termini. Change your termini sequences.")
		return None
	if print_messages: print(df)
	
	df = df.sort_values(by=["query acc.ver"]) #This ensures that the C terminus will always be first [0, "s.start"]
	df = df.set_axis([0, 1], axis=0, copy=False)
	
	if (len(df) != 2):
		if print_messages: print(df)
		if print_messages: print("There wasn't enough to match the C and N termini. Change your termini sequences.")
		return None
	elif ((df.at[0, "subject acc.ver"]) != (df.at[1, "subject acc.ver"])):
		if print_messages: print(df)
		if print_messages: print("There wasn't enough to match the C and N termini. Change your termini sequences.")
		return None
	
	contig = df.at[0, "subject acc.ver"]
	
	if bool(df.loc[0, "s. start"] < df.loc[1, "s. start"]): #C terminus is before N terminus. The orientation is reversed.
		if bool(df.loc[0, "s. end"] < df.loc[0, "s. start"] < df.loc[1, "s. end"] < df.loc[1, "s. start"]): #C terminus is before N terminus - negative strand
			start = int(df.at[0, "s. start"]) - flank
			end = int(df.at[1, "s. end"]) + flank	
			caps = {'first':start, 'last':end, 'contig':contig, "correct_ori":False, 'termini':termini}
		else:
			if print_messages: print(df)
			if print_messages: print("The orientation was wrong.")
			return None
	elif bool(df.loc[0, "s. start"] > df.loc[1, "s. start"]): #the N terminus is before the C terminus - correction orientation
		if bool(df.loc[1, "s. start"] < df.loc[1, "s. end"] < df.loc[0, "s. start"] < df.loc[0, "s. end"]): #N terminus is before C terminus - positive strand
			start = int(df.at[1, "s. start"]) - flank #This should be pulling the N terminus and storing it in start
			end = int(df.at[0, "s. end"]) + flank
			caps = {'first':start, 'last':end, 'contig':contig, "correct_ori":True, 'termini':termini}
		else:
			if print_messages: print(df)
			if print_messages: print("The orientation was wrong.")
			return None	
	else:
		if print_messages: print("something went terribly wrong")
	
	return(caps)

def reverse_complement(seq):
	new_seq = ""
	complement = {'A':'T', 'C':'G', 'G':'C', 'T':'A', 'N':'N', 'n':'n', 'a':'t', 't':'a', 'c':'g', 'g':'c'}
	for nuc in seq:
		new_seq = new_seq + str(complement.get(str(nuc)))
	return new_seq[::-1]

def get_termini_list(b_result):
	termini_list = []
	with open(b_result) as file:
		for line in file:
			if not line.startswith("#"):
				termini = line.split("\t")[0].replace('_n_terminus', '').replace('_c_terminus', '')
				termini = termini.replace('_n_terminal', '').replace('_c_terminal', '')
				termini = termini.replace('_n_term', '').replace('_c_term', '')
				if termini not in termini_list:
					termini_list.append(termini)
	return termini_list

def combine_caps(caps1, caps2):
	if caps1 == caps2:
		return caps1

	if caps1['correct_ori'] == caps2['correct_ori'] and caps1['contig'] == caps2['contig']:
		window1 = caps1['last'] - caps1['first']
		window2 = caps2['last'] - caps2['first']

		if window1 > window2:
			return caps1
		else:
			return caps2
	
	else:
		print("something went terribly wrong")
		exit()


if (not os.path.isfile(b_result)):
	print("No blast result file")
	exit()

termini_list = get_termini_list(b_result)

all_caps = {}
for termini in termini_list:
	caps = gene_ext(b_result, termini, flank)
	if caps is not None:
		print(caps)
		if caps['contig'] not in all_caps:
			all_caps[caps['contig']] = caps
		else:
			all_caps[caps['contig']] = combine_caps(caps, all_caps[caps['contig']])

if (os.path.isfile(genome_name + "_gene.fa")):
	os.system(genome_name + "_gene.fa")
print("\nFinal:")
for contig, caps in all_caps.items():
	print(caps)
	check_flank(caps)
	write_file(caps)
```

Run the python script:
```
python3 gene_ext.py arcto_2_1
```
If it says "Gene extracted" at the bottom, you're good to go! You can also check to make sure you have a file called `<genome_name>_gene.fa` (ex: `arcto_2_1_gene.fa`).

### Run augustus to get a preliminary annotation

Install Augustus or use the following module and conda environment (available to all with access to the `fslg_pws472` folder).
```
source /grphome/fslg_pws472/.bashrc
conda activate busco
```
(That's not a typo, the conda environment has BUSCO installed, which includes Augustus so it works for our purposes :) ).

Use your preferred text editor (vim, vi, nano) to create a text file and name it `annotation.sh`
```
nano annotation.sh
```

Copy and paste the job script below into your job file, editing it to include your email.
```
#!/bin/bash

#SBATCH --time=24:00:00   # walltime
#SBATCH --ntasks=1   # number of processor cores (i.e. tasks)
#SBATCH --nodes=1   # number of nodes
#SBATCH --mem-per-cpu=4G   # memory per CPU core
#SBATCH -J "augustus"   # job name
#SBATCH --mail-user=ashlynpowell913@gmail.com   # email address
#SBATCH --mail-type=BEGIN
#SBATCH --mail-type=END
#SBATCH --mail-type=FAIL
# LOAD MODULES, INSERT CODE, AND RUN YOUR PROGRAMS HERE

species=$1
gene="${species}_gene.fa"
output="${species}.gff"

source /grphome/fslg_pws472/.bashrc
conda activate busco

export AUGUSTUS_CONFIG_PATH=/grphome/fslg_pws472/apps/miniconda3/envs/busco/config/
augustus --strand=both --singlestrand=true \
--extrinsicCfgFile=./extrinsic.cfg \
--alternatives-from-evidence=true \
--gff3=on \
--uniqueGeneId=true \
--UTR=off \
--species=fly \
$gene > \
$output
```

Now submit the job script with `sbatch`. Make sure to give it the name of your genome when you run it (without the `.fasta`). For example:
```
sbatch annotation.sh arcto_2_1
```
You can check that it's running with:
```
squeue -u <your_username>
```

When it's done you can download the `.gff` and `gene.fa` files to manually check the annotation in Geneious.
