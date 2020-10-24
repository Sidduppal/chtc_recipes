# Running unicycler on CHTC

[Unicycler](https://github.com/rrwick/Unicycler) is an assembling tool which basically uses a collection of different tool including SPades. It was original meant to 
work only on single genomes but this [tweet](https://twitter.com/ines_cim/status/1288208862445699074) showed that Unicycler is actually able to assemble metagenomes pretty well as well.
<br> It has its own caveats and tweaks but generally it is pretty straight forward to run.

## Transferrig files

Trim the raw reads beforehand (preferably using Trimmomatic), and then transfer them to your `/staging/<USER_NAME>` directory.

## Creating submit file

Make a directory to store your submit and bash files and use a text editor to create the following file:

```bash
job = gw_forcepia_metagenome_spades_err_corr_unicycler
universe = vanilla
log = $(job)_$(Cluster).log
executable = /home/suppal3/guojun_project/unicycler_metagenome_reads/spades_corrected_unicycler/run_spades_unicycler.sh
#arguments = $(Process)
output = $(job)_$(Cluster)_$(Process).out
error = $(job)_$(Cluster)_$(Process).err
should_transfer_files = YES
when_to_transfer_output = ON_EXIT
#transfer_output_files =
request_cpus = 18
request_memory = 400GB
request_disk = 300GB
requirements = (Target.HasCHTCStaging == true)
queue 1
```
NOTE: If you are using a complex metagenome like sponge or soil, preferably give it large memory (like I have done).

## Creating bash file

By default SPades has memory limit of 250Gb ([See](http://cab.spbu.ru/files/release3.12.0/manual.html),  `Ctrl+F` for `m`), and most memory is used during the error correction
step. When you are running just Spades, you can adjust the memory limit using the `-m` flag. However, since unicycler was originally build to assemble, single
genomes (in which case so much memory is not needed), it does not have such an option. An alternative approach is to run error correction step of SPades, and then
provide the error corrected reads to unicycler using the `--no_correct` flag so that it know to skip the error correction step. [See](https://github.com/rrwick/Unicycler/issues/117)


```bash
#!/bin/bash

# Setting up the conda environment
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
export HOME=$PWD
export PATH
sh Miniconda3-latest-Linux-x86_64.sh -b -p $PWD/miniconda3
export PATH=$PWD/miniconda3/bin:$PATH
rm Miniconda3-latest-Linux-x86_64.sh

conda init bash
source ~/.bashrc

conda install -y -c bioconda python=3.6
conda install -y -c bioconda unicycler
conda install -y -c bioconda samtools openssl=1.0
conda install -y -c bioconda spades

mkdir gw_forcepia_unicycler_output

# Transfer the reads from your staging directory
rsync -azP /staging/suppal3/guojun_project/GW_Forcepia_1P.fastq.gz ./
rsync -azP /staging/suppal3/guojun_project/GW_Forcepia_2P.fastq.gz ./

# Run error correction step of SPades
spades.py -1 GW_Forcepia_1P.fastq.gz -2 GW_Forcepia_2P.fastq.gz -o spades_error_correected_reads -t 18 -m 400 --only-error-correction

rm GW_Forcepia_1P.fastq.gz
rm GW_Forcepia_2P.fastq.gz

cd spades_error_correected_reads/corrected/

gunzip *.gz

cd ../../

# Run unicycler with --no_correct flag
unicycler --short1 spades_error_correected_reads/corrected/GW_Forcepia_1P.fastq.00.0_0.cor.fastq --short2 spades_error_correected_reads/GW_Forcepia_2P.fastq.00.0_0.cor.fastq --out gw_forcepia_unicycler_output --no_correct  --threads 18

tar czvf spades_error_correected_reads.tar.gz spades_error_correected_reads
tar czvf gw_forcepia_unicycler_output.tar.gz gw_forcepia_unicycler_output

mv spades_error_correected_reads.tar.gz /staging/suppal3/guojun_project/
mv gw_forcepia_unicycler_output.tar.gz /staging/suppal3/guojun_project/

rm *.fastq
```
