Running GTB-Tk on CHTC
======================
[GTDB-Tk](https://github.com/Ecogenomics/GtdbTk) is a pipeline to identify genomes based on protein markers, using a very large reference bacterial tree of life. The taxonomy used was constructed as a huge genome tree, with fewer polyphyletic groups, and consistent evolutionary divergence (see [this paper](https://www.nature.com/articles/nbt.4229) and the [Genome Taxonomy Database](http://gtdb.ecogenomic.org/)). 

Setting up the necessary software
---------------------------------
* Log into your CHTC submit
* Create a directory to hold your build and submit files, then `cd` to that directory
* Use a text editor to create a file called `build.sub` with the following contents:

```bash
universe = vanilla
log = interactive.log

output = process.out
error = process.err

+IsBuildJob = true
requirements = (OpSysMajorVer =?= 7) && (IsBuildSlot == true)

should_transfer_files = YES
when_to_transfer_output = ON_EXIT

request_cpus = 1
request_memory = 4GB
request_disk = 10GB

queue
```

* Now start an interactive job with the following command:

```bash
condor_submit -i build.sub
```

* The first step is to download and compile Python 3.7.9:

```bash
wget https://www.python.org/ftp/python/3.7.9/Python-3.7.9.tgz
tar xvf Python-3.7.9.tgz
cd Python-3.7.9
./configure --prefix=${PWD}/../python
make
make install
cd ..
export PATH=${PWD}/python/bin:$PATH
cd python/bin
#telling bash to use python3 instead of 2. Python3 automatically installs pip3 as well
alias python=${PWD}/python3
#check python version. It should be python3 to be able to use pip3
python -V
cd ../..
```

* Now we install gtdbtk:

```bash
pip3 install gtdbtk future numpy
```

* Then we install some other programs that gtdbtk needs:

```bash
wget https://github.com/hyattpd/Prodigal/releases/download/v2.6.3/prodigal.linux
chmod +x prodigal.linux
mv prodigal.linux python/bin/prodigal
```

```bash
wget http://eddylab.org/software/hmmer/hmmer-3.2.tar.gz
tar xvf hmmer-3.2.tar.gz
cd hmmer-3.2
./configure
make
find src -perm /a+x -exec cp {} ../python/bin \;
cd ..
rm -rf hmmer-3.2*
```

```bash
wget https://github.com/matsen/pplacer/releases/download/v1.1.alpha19/pplacer-linux-v1.1.alpha19.zip
unzip pplacer-linux-v1.1.alpha19.zip
cd pplacer-Linux-v1.1.alpha19/scripts
python setup.py install
cd ..
cp guppy ../python/bin/
cp pplacer ../python/bin/
cp rppr ../python/bin/
cd ..
rm -rf pplacer-*
```

```bash
wget https://github.com/ParBLiSS/FastANI/releases/download/v1.1/fastani-Linux64-v1.1.zip
unzip fastani-Linux64-v1.1.zip
mv fastANI python/bin/
rm fastani-Linux64-v1.1.zip
```

```bash
wget http://www.microbesonline.org/fasttree/FastTree
wget http://www.microbesonline.org/fasttree/FastTreeMP
chmod +x FastTree*
mv FastTree* python/bin/
```

Before we go on we need to change the first line of a few files in the python installation. The reason is that they will have hard-coded the current absolute path.

* First find the absolute path to your directory with the pwd command, then use the commands below to find all files that mention it and swap it for a general path (below we use the path /var/lib/condor/execute/slot1/dir_5943 as an example - make sure to substitute this for your directory path):

```bash
grep -r -I /var/lib/condor/execute/slot1/dir_5943 ./python/* | cut -f 1 -d ':' | sed "s?^?sed -i 's;#!/var/lib/condor/execute/slot1/dir_5943/python/bin/python;#!/usr/bin/env python;' ?" > sed.sh
chmod +x sed.sh
./sed.sh
rm sed.sh
```

* Before finishing, you need to compress the `python` directory:

```bash
tar -czvf gtdbtk_python.tar.gz python
```

* Then finish the interactive job with the `exit` command. If everything worked correctly, the job should copy over the `gtdbtk_python.tar.gz` file to the submit node.

Before you run GTDB-Tk, you must download the lastest GTDB-tk database (r_95 as of 2020) a large (27 GB) file to your `/staging/<USERNAME>` folder.

```bash
cd /staging/<USERNAME>
wget https://data.ace.uq.edu.au/public/gtdb/data/releases/release95/95.0/auxillary_files/gtdbtk_r95_data.tar.gz
```

Running GTDB-Tk on CHTC
-----------------------
* Create a directory to hold your job files and `cd` into that directory
* Create a subdirectory to hold your genome fasta files. We'll assume it is called `input_fasta_files`. Prefrebly try to create this folder in `/staging/<USERNAME>`. I have noticed that when all the files are transferred from `/staging/<USERNAME>` the chances of crashing due to pplacer are less.
* Copy over your input files to the `input_fasta_files` directory
* Now use a text editor to make a submit file called `gtdbtk.sub`, with the following contents:

```bash
universe = vanilla
log = gtdbtk_$(Cluster).log
error = gtdbtk_$(Cluster)_$(Process).err
output = gtdbtk_$(Cluster)_$(Process).out
executable = run_gtdbtk.sh
should_transfer_files = YES
when_to_transfer_output = ON_EXIT
transfer_input_files = /staging/suppal3/gtdbtk_python3.tar.gz, /staging/suppal3/gtdbtk_r95_data.tar.gz, /staging/suppal3/input_fasta_files
#Reason for using a single CPU : https://github.com/Ecogenomics/GTDBTk/issues/124#issuecomment-492440700 
request_cpus = 1
request_memory = 400GB
request_disk = 250GB
requirements = (Target.HasCHTCStaging == true)
queue 1
```

Note: Remember to adapt the exact path to the `tar.gz` files to fit your setup.

* Use a text editor to make a bash script to set up and run the job called `run_gtdbtk.sh` with the following contents:

```bash
#!/usr/bin/bash

# Expand gtdbtk files
tar xf gtdbtk_python3.tar.gz
rm gtdbtk_python3.tar.gz
tar xf gtdbtk.r95_data.tar.gz
rm gtdbtk.r95_data.tar.gz

# Set up path to executables and python etc.
export PATH=${PWD}/python/bin:$PATH
export GTDBTK_DATA_PATH=${PWD}/release86
sed -i "1s|.*|#!${PWD}/python/bin/python3|" ${PWD}/python/bin/gtdbtk

# Run GTDB-Tk
gtdbtk classify_wf --genome_dir input_fasta_files --extension fasta --out_dir gtdbtk_output --cpus 1

# Compress output
tar -czvf gtdbtk_output.tar.gz gtdbtk_output
```

Note: Remember to alter the gtdbtk arguments so that the number of CPUs requested match the `.sub` file. The `--extension` argument specifies the extension of files to use in the genome directory (here we specify `.fasta`). You can also ty to use the `--scratch_dir` argument which tells the pipeline to use more disk vs. memory. Removing `--scratch_dir` will make the pipeline faster, but it will need ~90 GB of RAM.

* Make the script executable:

```bash
chmod +x run_gtdbtk.sh
```

* Now submit the job and :pray: :bowtie: 

```bash
condor_submit gtdbtk.sub
```
