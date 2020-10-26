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
