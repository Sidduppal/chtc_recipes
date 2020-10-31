# Running nonpareil on CHTC

## Creating submit file

Make a directory to store your submit and bash files and use a text editor to create the following file:

```bash
universe = vanilla
log = nonpareil_$(Cluster).log
error = nonpareil_$(Cluster)_$(Process).err
output = nonpareil_$(Cluster)_$(Process).out
executable = run_nonpareil.sh
should_transfer_files = YES
when_to_transfer_output = ON_EXIT

#transfer_input_files = 

request_cpus = 16
request_memory = 300GB
request_disk = 1200GB
requirements = (Target.HasCHTCStaging == true)
queue 1
```
Make sure the memory is higher than the size of the metagenome you are analysing. See documentation [here](https://nonpareil.readthedocs.io/en/latest/redundancy.html#common-options)

## Creating bash file

Whenever you are doing this be very mindful of the disk usage. You fill need about 4x the amount of disk space (in Gb) than the number of bases in the metagenome. Like the [this SRA](https://www.ncbi.nlm.nih.gov/sra/?term=SRR10389008) has about 200G bases. When it is downloaded as `fastq` its size would be around 210Gb. Mutiply this by two as there would be forward as well as reverse reads. Then you'll trim them which would again double their size. So for safety I sometimes even go 5x the disk space (in Gb) of the number of bases.

Use a text editor to create the following file:

```bash
# Set up conda and install dependencies

wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
export HOME=$PWD
export PATH
sh Miniconda3-latest-Linux-x86_64.sh -b -p $PWD/miniconda3
export PATH=$PWD/miniconda3/bin:$PATH
rm Miniconda3-latest-Linux-x86_64.sh

conda init bash # needed to restart bash from a shell script
source ~/.bashrc

# Good for debugging
conda info -a

conda install -y -c bioconda sra-tools
conda install -y -c bioconda trimmomatic
conda install -y -c bioconda nonpareil

# You can directkly download the `fastq` files from SRA or download the compressed `sra` files and then uncompress them into `fastq`. 
# Download fastq files from sra
fastq-dump -I --split-files SRR352287

# You can also set up a while loop to download all the fastq files together, I don't recommend it as then there are higher chances of the job terminating due 
to it running out of disk space. I prefer to download the fastw file, trim it, calculate the coverage by running nonpareil and then delete it before downloading 
any other fastq files.
# Script to download all fastq files together. 
#cat SRR_numbers.txt | while read line; do fastq-dump -I --split-files ${line}; done

# Pre-rpocessing of the reads
# Important to remove any read that has length below 25, as that can cause this [error](https://github.com/lmrodriguezr/nonpareil/issues/37)

trimmomatic PE -threads 16 -baseout SRA_reads.fastq SRR352287_1.fastq SRR352287_2.fastq \
ILLUMINACLIP:$PWD/miniconda3/pkgs/trimmomatic-0.39-1/share/trimmomatic/adapters/TruSeq3-PE.fa:2:30:10 MINLEN:25

# Run nonpareil
nonpareil -s SRA_reads_1P.fastq -T kmer -f fastq -b SRR352287 -t 16 -R 300000

rm *.fastq

# Now you can download the process to analyze more metagenomes in a single bash file, rather than submitting multiple jobs.
```
Here SRR_numbers.txt looks like:
SRR10406097   
SRR10406098   
SRR10406099   
SRR10406100   
SRR10406101   
SRR10406102   

# Concatenating multiple runs together

In case you SRA reads have multiple runs, like with [this SRA](https://www.ncbi.nlm.nih.gov/sra/?term=SRR398109). This SRA ID has two sets of reads, which would probably be becasue they ran it the second time to get a higher coverage.
<br> We first download the two sets of reads separately and then trim them. Then we concatenate them together.

```bash
fastq-dump -I --split-files SRR398109

trimmomatic PE -threads 16 -baseout SRA_reads_SRR398109.fastq SRR398109_1.fastq SRR398109_2.fastq \
ILLUMINACLIP:$PWD/miniconda3/pkgs/trimmomatic-0.39-1/share/trimmomatic/adapters/TruSeq3-PE.fa:2:30:10 MINLEN:25

fastq-dump -I --split-files SRR398111

trimmomatic PE -threads 16 -baseout SRA_reads_SRR398111.fastq SRR398111_1.fastq SRR398111_2.fastq \
ILLUMINACLIP:$PWD/miniconda3/pkgs/trimmomatic-0.39-1/share/trimmomatic/adapters/TruSeq3-PE.fa:2:30:10 MINLEN:25

# Concatenating the reads together
cat SRA_reads_SRR398109_1P.fastq  SRA_reads_SRR398111_1P.fastq > SRA_reads_1P.fastq

# To check if the number of lines are divisible by four
echo $(cat SRA_reads_1P.fastq | wc -l)/4 | bc

nonpareil -s SRA_reads_1P.fastq -T kmer -f fastq -b SRX115467 -t 16 -R 300000

rm *.fastq
```
Whenever you are concatenating stuff beware that the amount of disk space needed would increase as well. So take that into consideration while making the bash file.

### Update Oct 31st, 2020 

A newer and much much faster way to download the reads is to use `fasterq-dump` instead of `fastq-dump`. This is an updated version of `fastq-dump`.
<br>[see documentation](https://github.com/ncbi/sra-tools/wiki/HowTo:-fasterq-dump)
It is exactly like `fastq-dump` but better and faster. It can also use multiple threads. Its default behaviour is to use `split 3` which creates three fastq files. 
<br>Why use split 3 - [this](https://www.biostars.org/p/186741/) and [this](https://www.biostars.org/p/156909/).  First two are for fwrd and rev reads while the other one is for unpaired reads. It also trims reads with length less than 20bp. 
<br> However, a note of caution here is that `fasterq-dump` uses much more disk space. [See](https://www.biostars.org/p/156909/), they advise to have about 8x to 10x the size of accession to be available as free space in your laptop. 
<br> You can see the size of an accession using `vdb-dump --info SRR341578`. 
<br> Multiple threads can be used using `-e` flag
<br> However, this command is only available on versions > 2.9 and you need to install python 3.7 to use it. Here is the bash file that I use for a simple accession pull:

```bash
#!/usr/bin/bash

# Set up conda and install dependencies

wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
export HOME=$PWD
export PATH
sh Miniconda3-latest-Linux-x86_64.sh -b -p $PWD/miniconda3
export PATH=$PWD/miniconda3/bin:$PATH
rm Miniconda3-latest-Linux-x86_64.sh

conda init bash # needed to restart bash from a shell script
source ~/.bashrc

# Good for de-bugging
conda info -a 
conda install -y -c bioconda python=3.7
conda install -y -c bioconda sra-tools=2.10
conda install -y -c bioconda trimmomatic
conda install -y -c bioconda nonpareil

fasterq-dump SRR9009774 -e 16

trimmomatic PE -threads 16 -baseout SRA_reads.fastq SRR9009774_1.fastq SRR9009774_2.fastq \
ILLUMINACLIP:$PWD/miniconda3/pkgs/trimmomatic-0.39-1/share/trimmomatic/adapters/TruSeq3-PE.fa:2:30:10 MINLEN:25

nonpareil -s SRA_reads_1P.fastq -T kmer -f fastq -b SRR9009774 -t 16 -R 300000

rm *.fastq
```

