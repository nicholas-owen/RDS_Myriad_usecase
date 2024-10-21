# Quick Myriad use-case
#### N Owen 2024
## UCL Cluster
A cluster is a collection of computers that are used for a wide range of purposes, including scientific research and data analysis. Using many computers at once allows users to handle big amounts of data and run processes that require a lot of computing power and that otherwise would not be possible to run on a single computer. Importantly, many cluster have scheduling systems that can manage and allocate computing resources efficiently among users and applications.


## Register
Register for an account on Myriad at: https://www.rc.ucl.ac.uk/docs/Account_Services/

## New User Guide
https://www.rc.ucl.ac.uk/docs/New_Users/

## Software
Myriad uses a module system to manage software. ​It means that a lot of the software that you might need to run your analyses has already been installed, and can be loaded and unloaded as needed. This is extremely helpful as ensures that users can access the specific tools and versions they need for their research without having to install them themselves.

A brief introduction on the general use of Myriad modules is [here](https://www.rc.ucl.ac.uk/docs/Software_Guides/Other_Software/) , and a full list of modules currently installed is [here](https://www.rc.ucl.ac.uk/docs/Installed_Software_Lists/module-packages/). 


## Data
Create a directory for the data we are going to download.

```
mkdir -p RNA_Seq/data
```

What does the -p flag do? Now, go to the `data` directory you have created.

```
cd RNA_Seq/data
```

Now that we're in the correct directory, we will use curl to download some bulk RNA-Seq data.


```
curl http://data.biostarhandbook.com/rnaseq/projects/griffith/griffith-data.tar.gz --output griffith-data.tar.gz
```

Let's take a look at this Unix command line. The `curl` command is used to retrieve data from web sites. A similar command is `wget`. The Unix system you are working with may have either `curl` or `wget` installed. 

Let's `untar` and `unzip` our file.


```
tar -xvf griffith-data.tar.gz
```

You should see each of the files listed as the `tar` is decompressed. Two directories were created in this process: a reads directory and a refs directory. In the reads directory there are 12 fastq files. In the refs directory, there are 4 files, containing genome and annotation information. Keep in mind that we will be using a subsetted reference file from human chromosome 22.

The `fastq` files are unzipped, but you may obtain zipped fastq files in the future. Because many bioinformatics programs can work directly with fastq.gz files, let's compress these files to save space.


```
gzip reads/*.fq  
```

Note the use of the * wildcard. We are using gzip to zip all files ending in .fq in the directory reads.
To peek inside these files after zipping you can use zcat or gzcat (for a mac) paired with head. This works similar to cat paired with head


```
zcat reads/HBR_1_R1.fq | head -n 8
```

In this case, we are "piping" - with the pipe symbol |, the results of zcat into head and selecting the top 8 lines of the file (-n 8).

The results should show the top 8 lines of the `.fq.gz` file.


```
@HWI-ST718_146963544:7:2201:16660:89809/1
CAAAGAGAGAAAGAAAAGTCAATGATTTTATAGCCAGGCAAAATGACTTTCAAGTAAAAAATATAAAGCACCTTACAAACTAGTATCAAAATGCATTTCT
+
CCCFFFFFHHHHHJJJJJHIHIJJIJJJJJJJJJJJJIJJJJJJJJJJJJJIJJIIJJJJJJJJJJJJIIJFHHHEFFFFFEEEEEEEDDDDCDDEEDEE
@HWI-ST718_146963544:7:2215:16531:12741/1
CAAAATATTTTTTTTTTCTGTATATGACAAGACACACATCAGATCATAAGCTACAAGAAAACAAACAAAAAAGATATGAAAAAGATATAAAGACCTCCCC
+
@@@DDDDDFFFFFIIII;??::::9?99?G8;)9/8'787.)77;@==D=?;?A>D?@BDC@?CC=?BBBBB?<:4::@BBBB<?:>:@DD343<>:?BB
```


Keep in mind, there are several Unix commands that can be used to look at the contents of files, each has it's own flags/options and is used slightly differently. For example:

```
less
more
cat
head 
tail
```

less, in particular, can also be used to examine zipped files with the help of lesspipe, on certain unix systems. 

## FASTQC Quality Control of Data
Lets have a look at the quality of the FASTQ data. The program used is called [FASTQC](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/). This will analyse the FASTQ sequence data and provide summary statistics.

Firstly, lets load the module in Myriad: (tip: have a look for FASTQC in the modules list above.)

```
module load fastqc/0.11.8
```

and lets run it on one sequence file:

```
fastqc ./reads/HBR_1_R1.fq.gz
```
Check the summary out put to see what the software is doing.

## Myriad Job Submissions

The general format of a script to submit as a job to Myriad, like other clusters that use SGE, is below. UCL services use Grid Engine to manage jobs. After creating your script, submit it to the scheduler with:
Notes on script formatting [here](https://www.rc.ucl.ac.uk/docs/Experienced_Users/).

```
qsub my_script.sh
```

create a script file containing the following: `my_script_01.sh`

```
#!/bin/bash -l
#$ -cwd # use current working directory
#$ -o output.txt # output file
#$ -e error.txt # error log file
#$ -N Test_HPC_Job # name of the job
#$ -l h_rt=01:00:00 # maximum runtime 
#$ -l h_vmem=4G # memory per core
#$ -pe smp 1 # number of cores , if using one this isnt needed

# Unload conflicting modules
module unload gcc-libs

# Load necessary modules
module load fastqc/0.11.8

# Create the Results directory if it doesn't exist
RESULTS_DIR="$HOME/results"
mkdir -p "$RESULTS_DIR"

# Run the FASTQC command
fastqc HBR_*.fq UHR_*.fq -o "$RESULTS_DIR"
```

submit the script:

```
qsub my_script_01.sh
```
## Output
The results will be saved within a directory called `~/results/`

## Things to consider
We used one cluster job script to run one command over multiple files

Is this efficient use of the cluster ?

What happens with >1000 files ?

How else could we approach this ?

Hint: Cluster is built of numerous nodes..

How about ?

One script finds all the `fastq.gz` files, then creates a new script itself that runs the analysis on one file only, outputting the results. The original script then submits the new script as a new job. 1000 files analysis would then turn into 1000 cluster jobs with quicker priorities in the queue. Think on how queuing works.

For example: 

```
###FASTQC analysis of FASTQ files

if [ $run_mode = "--fastqc" ] ; then
    
    config_file=./project.config
    server_file=./server.config
    set -o allexport
    source $config_file
    source $server_file
    set +o allexport
    
    fastqc_files="$2"
    report_output="$3"
    
    if [ -z $fastqc_files ] ; then
        echo -e ""
        echo -e "${RED}No FASTQ files specified.${NC}"
        echo -e ""
        echo -e "     Please specify a plain text file listing all FASTQ samples to be assayed."
        echo -e "     Format: one fastq (or gz'd archive) per line"
        echo -e "     Usage: ${YELLOW}--fastqc <filename.ext> <report_dir_output>${NC}"
        echo -e ""
        exit 1
    fi
    
    if [ -z $report_output ] ; then
        echo -e ""
        echo -e "${RED}No report output directory specified.${NC}"
        echo -e ""
        echo -e "     Please specify a directory that will be used under ${reports_fastqc} for this analysis."
        echo -e "     Usage: ${YELLOW}--fastqc <report_dir_output>${NC}"
        echo -e ""
        
        exit 1
    fi
    
    echo -e "Creating report output directory: ${RED}${reports_fastqc}/${report_output}${NC}"
    
    mkdir "${reports_fastqc}/${report_output}"
    
    #bash ./cluster/scripts/prepare_fastqc_scripts.sh ${fastqc_files} ${report_output}
    
    mkdir ${reports_fastqc}/${report_output}
    while IFS='' read -r line
    do
        file_name="$line"
        echo "Filename read from file - $file_name"
        script_file=`echo $file_name | awk -F '/' '{print $NF}'`
        
        echo "Creating script: $script_file."
        echo "#$ -l h_vmem=3.9G" > ${cluster_scripts_loc}/fastqc_$script_file.sh
        echo "#$ -l tmem=3.9G" >> ${cluster_scripts_loc}/fastqc_$script_file.sh
        echo "#$ -l h_rt=2:0:0" >> ${cluster_scripts_loc}/fastqc_$script_file.sh
        echo "#$ -pe smp 1" >> ${cluster_scripts_loc}/fastqc_$script_file.sh
        echo "#$ -j y" >> ${cluster_scripts_loc}/fastqc_$script_file.sh
        echo "#$ -R y" >> ${cluster_scripts_loc}/fastqc_$script_file.sh
        echo "#$ -o ${cluster_output_loc}" >> ${cluster_scripts_loc}/fastqc_$script_file.sh
        echo "#$ -e ${cluster_output_loc}" >> ${cluster_scripts_loc}/fastqc_$script_file.sh
        echo "#$ -S /bin/bash" >> ${cluster_scripts_loc}/fastqc_$script_file.sh
        echo "export JAVA_HOME=${javaFolder}" >> ${cluster_scripts_loc}/fastqc_$script_file.sh
        echo "export PATH=$PATH:${javaFolder}:" >> ${cluster_scripts_loc}/fastqc_$script_file.sh
        echo "${fastqcFolder}/fastqc ${fastq_loc}/$file_name -o ${reports_fastqc}/$report_output" >> ${cluster_scripts_loc}/fastqc_$script_file.sh
        echo -e "Submitting job to cluster: ${YELLOW}fastqc_${script_file}.sh${NC}"
        qsub ${cluster_scripts_loc}/fastqc_$script_file.sh
    done < "$fastqc_files"
    
    
fi

```


## Extras

Can you write a script to use [MultiQC](https://multiqc.info/) to summarise the results of the FASTQC analysis ?