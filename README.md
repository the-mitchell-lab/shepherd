# SHEPHERD

Welcome to SHEPHERD! The explanation below is a brief summary of how to use SHEPHERD. Please report any bugs either via GitHub or by contacting the developers directly.

Lastly, I have one important note. IF YOU WANT TO KILL SHEPHERD/END THE PROCESS, USE Ctrl+C! Do not use Ctrl+Z or another combination, as these are not able to be processed properly and could result in [ZOMBIE PROCESSES](https://stackoverflow.com/questions/20688982/zombie-process-vs-orphan-process). 

## Requirements
* python (Most recently tested with v3.14)
* multiprocess
* scikit-learn
* gcc_linux-64
* biopython
* prodigal
* HMMER
* networkx
* matplotlib
* hdbscan
* pandas

Additionally, some features specific to RiPP classification rely on legacy software below
* [Meme Suite](https://meme-suite.org/meme/)
* [RREFinder](https://github.com/Alexamk/RREFinder)
* [RADAR](https://github.com/AndreasHeger/radar)
* [fasta2](https://fasta.bioch.virginia.edu/wrpearson/fasta/fasta2/)

## Installation

1. Pull the git repository down onto your computer.
2. In the hmm_dir folder of the repo, press your Pfam-A file (press TIGRFAM too if desired)
    * If you are worried about space, you can edit the PFAM_DIR variable in the general section of the `confs/default.conf` file.

Note: If you are a MacOS user, if you are getting a "urlopen error [SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed..." when running SHEPHERD, please go to your applications folder and go to your Python 3 folder and run the "Install_Certificates.command" to fix this issue.

There may be things missing. Please let me know.

## General usage
I will run through a few examples and explain what they mean.
1. Basic SHEPHERD run. 
    *  `python shepherd.py query`
    *  SHEPHERD will run on query, and output will be in a folder titled `[query]_rodeo_out`
    *  `python shepherd.py BAL72546` for example, will run SHEPHERD on the BAL72546 accession and output results to `BAL72546_rodeo_out`
    *  `python shepherd.py lassos.txt` will run on every accession in `lassos.txt` and output results to `lassos_rodeo_out`. Notice that the file extension is ignored when naming the output folder.
2. Basic SHEPHERD with named output.
    * Say you wanted output in a particular folder, for example, `my_output`.
    * `python shepherd.py lassos.txt -out my_output`
    * Output will appear in `my_output_rodeo_out`.
3. SHEPHERD with custom HMMs.
    * SHEPHERD by default requires Pfam-A for HMM scanning. However, some RiPP heuristics make use of TIGRFAM. If you'd like to run TIGRFAM or another custom HMM, then use the `-hmm` or `--custom_hmm` flag with the path to your HMM. Note that you can input a list of HMMs as you will see below.
    * `python shepherd.py lassos.txt -hmm TIGRFAM.hmm MYFAVHMM.hmm` will use Pfam-A in addition to TIGRFAM and MYFAVHMM. Note that this syntax works only if the hmms are in the top level directory, as these are relative paths to shepherd.py.
4. Running in parallel
    * If you have a list of accessions and want to run SHEPHERD on them in parallel, use the `-j` or `--num_cores` flag followed by the number of processes you want to spawn. Note that it doesn't make sense to spawn more processes than your computer has CPUs. It also doens't make sense to spawn 4 processes if there are only 3 queries in the input file. What's the 4th process going to do?
    * `python shepherd.py lassos.txt -j 4`
    * NOTE: The output may not be in the same order as the input. This is because each process will write its output as soon as possible. Let me know if this is a serious inconvenience!
5. Configuration files.
    * SHEPHERD by default has a configuration file in `confs/` named `default.conf`. Take a look at this to get a feel for the syntax. 
    * The use of the conf file is to specify command line arguments in a file, provide colors for ORF diagrams, and make specific parameters for different types of RiPPs. If you make your own conf file in the `confs` folder, any parameter you specify will overwrite the corresponding one in the default configuration, however anything you don't specify will just use the default config value. You can supply multiple config files the same way that you supply multiple HMMs. Say you provide the flag `--conf_file confs/conf1 confs/conf2`, the parameters in conf2 take precedence over those in 1, and the parameters in conf1 take precedence over those in the default conf.
    * See the Configuration section for a more detailed description of syntax.
    * `python shepherd.py lassos.txt --conf_file confs/myconf.conf`

## Large-scale example of SHEPHERD usage
I will list the steps used for mining of RiPP gene clusters from all Acidobacterial assemblies in NCBI.
1. First, a list of all assemblies was gathered from the NCBI Datasets website. Here is a link to all [Acidobacterial assemblies](https://www.ncbi.nlm.nih.gov/datasets/genome/?taxon=57723).
   
    * The full table was downloaded.
    * Assemblies were dereplicated to avoid double-counting between GenBank and RefSeq data.
    * The list of assembly accession IDs was copied to a .txt file. Each line contained an identifier starting with "GCA_" or "GCF_".

2. Acidobacterial assemblies were then downloaded using the [NCBI Datasets command line tools](https://www.ncbi.nlm.nih.gov/datasets/docs/v2/getting_started/).
```
    datasets download genome accession --inputfile acido_accessions.txt --dehydrated --filename acido_data.zip
    unzip acido_data.zip -d acido_data
    datasets rehydrate --directory acido_data/
```

3. A bash script was written to run SHEPHERD on a single assembly.
   
    * This is one option for parallelizing the analysis, but the best option will vary based on your applications.
    * This script is adapted specifically to a specific personal PC, so it will need to be verified independently on your machine before entering large-scale analyses.
  
```
    #!/bin/bash
    filename="${1##* }";
    accessionid=$(cut -d'/' -f3 <<< $filename);
    mkdir -p ./acido/acido_output/${accessionid}/;
    echo -e "${accessionid}\tacido/acido_data/${filename}" > .//acido//acido_input//${accessionid}_run.txt;
    python3 shepherd.py ./acido/acido_input/${accessionid}_run.txt -out ./acido/acido_output/${accessionid}/ -j 2 -v -meta -bait ./hmm_dir/Metabait.hmm -hmm ./hmm_dir/TIGR*.hmm ./hmm_dir/RREF*.hmm --megarun 2> ./acido/acido_output/${accessionid}_log.txt;
    rm ./acido/acido_input/${accessionid}_run.txt;
    echo "Finished processing ${filename}";
```


4. Begin SHEPHERD analysis!

    * This command will enter the list of Acidobacterial assemblies for sequential multiprocessing using parallel.
    * Adapt this approach depending on compute resources.
    * The progress and joblog flags allow for monitoring and restarting. This may be necessary for large runs on personal machines like this one.
    * More sophisticated job handlers (e.g. SLURM) obviate many of these requirements. Again, your mileage may vary.

```
    cat acido.txt/acido_accessions.txt | parallel -j 8 --progress --joblog acido.log ./acido_submit.sh {}
```

5. Perform comparative genomics analysis using HERD.
   
    * Once finished, you will have a massive set of all relevant BGCs containing hits from your list of bait HMMs.
    * This can be analyzed holistically or sliced into relevant subsections.
    * The command below will analyze all of the Acidobacterial annotation outputs for loci containing both PF13471 (lasso peptidase) and PF00733 (lasso cyclase).
    * HERD has flags to change the depth of HMMs analyzed per gene and in total.
    * Numerous output files will be generated to facilitate downstream analysis.
    
```
    python3 cassette_reconstruction.py "/path/to/shepherd/acido/acido_output/*/main_co_occur.csv" Acido_lasso -f PF13471 PF00733
```


### Configuration syntax
Configurations should be placed in the conf folder. The syntax is as follows for each RiPP type. Note that the variable type (int or bool) only needs to be specified if it is not a string. If you are curious about what variable names are available, most command line arguments of SHEPHERD are. Also note that parameters specific to genbank file mining should go in the 'general' section, as the genbank files are mined the same no matter the RiPP type, as it is a preproccessing step.
```
>[Ripp_type or 'general']
#BEGIN_VARIABLES
[variable_type] VARNAME1 VALUE1
...
...
[variable_type] VARNAMEn VALUEn
#END_VARIABLES

#BEGIN_COLORS
HMM_ANNOTATION_ID1 [color1]
...
...
HMM_ANNOTATION_IDm [colorm]
#END_COLORS
>>
```
### More Notes (some redudancy)
1. Output is currently not verbose (You will not see all debug output). For those of you who would like to see it, uncomment line 16 and comment line 17 in `shepherd.py`
2. Output is not in order if ran in parallel. Output should still make sense but the accessions might not be in the same order due to parallel processing.
3. You may need to add an email and API_KEY to the entrez_utils.py file.
