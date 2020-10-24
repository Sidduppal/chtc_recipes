# Running GTB-Tk on CHTC

[GTDB-Tk](https://github.com/Ecogenomics/GtdbTk) is a pipeline to identify genomes based on protein markers, using a very large reference bacterial tree of life. The taxonomy used was constructed as a huge genome tree, with fewer polyphyletic groups, and consistent evolutionary divergence (see [this paper](https://www.nature.com/articles/nbt.4229) and the [Genome Taxonomy Database](http://gtdb.ecogenomic.org/)). 

* Log into your CHTC submit
* `cd` into your `/staging/<USER_NAME>` and create a directory to hold your genome fasta files. Let's call this directory, `input_fasta_files`. Your work in staging is now done. `cd` into your `home` directory.

## Creating your submit file

* Create a directory to hold your build and submit files, then `cd` to that directory
* Use a text editor to create a file called `gtdbtk.sub` with the following contents:

```bash
universe = vanilla
log = gtdbtk_$(Cluster).log
error = gtdbtk_$(Cluster)_$(Process).err
output = gtdbtk_$(Cluster)_$(Process).out
executable = run_gtdbtk.sh
should_transfer_files = YES
when_to_transfer_output = ON_EXIT
transfer_input_files = /staging/groups/kwan_group/gtdbtk_data.tar.gz, /staging/suppal3/input_fasta_files

#Reason for using a single CPU : https://github.com/Ecogenomics/GTDBTk/issues/124#issuecomment-492440700 
request_cpus = 1

request_memory = 400GB
request_disk = 250GB
requirements = (Target.HasCHTCStaging == true)
queue 1
```
## Creating your bash file

* Use a text editor to create a file called `run_gtdbtk.sh` with the following contents:
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

tar xf gtdbtk_data.tar.gz
rm gtdbtk_data.tar.gz

conda install -y -c bioconda python=3.7
conda install -c conda-forge -c bioconda gtdbtk -y

echo "export GTDBTK_DATA_PATH=${PWD}/release95/" > miniconda3/etc/conda/activate.d/gtdbtk.sh

source ~/.bashrc

# Run GTDB-Tk
gtdbtk classify_wf --genome_dir input_fasta_files --extension fasta --out_dir gtdbtk_output --cpus 1

# Compress output
tar -czvf gtdbtk_output.tar.gz gtdbtk_output
```

Great job! Everything looks good, make sure to double check the paths and when you are ready submit the job using, `condor_submit gtdbtk.sub`
Now wait and pray (seriosly pray) :pray:

