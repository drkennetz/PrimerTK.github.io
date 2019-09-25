# Standard and Multiplex Tutorial

These two pipelines have very similar workflow so I will include them together. If you followed the installation page and git cloned the repo, there is a directory called `test` in the PrimerTK repo. This is for code testing but it also has data for the user to implement the full pipeline (testing is good).

If you added primer3 and isPcr to you .bashrc in the installation guide, skip this. If not, run the following (assuming you installed primer3 and isPcr in your `~/bin` directory:

```
export PATH=~/bin/primer3:$PATH
export PATH=~/bin/isPcr33:$PATH
```

Or add the above to your `~/.bashrc` to prevent yourself from having to do this every time you log in to a new session.
So now `primer_tk`, `isPcr`, and `primer3_core` should all be accessible from the command line, anywhere in your system.
This tutorial is going to assume that you installed PrimerTK in your home directory but you may well have set it up anywhere. It is also going to assume you used the standard installation script so the path to the primer3 executable and isPcr executable will reflect that.

## 1) Setup a tutorial directory with files from test

These files are small so you can copy them over directly or just point to them as you run the program.

I will do everything in a directory called tutorial:

```
cd ~/tutorial/
cp ~/PrimerTK/test/data/humrep.ref .
cp ~/PrimerTK/test/data/input_standard.csv .
cp ~/PrimerTK/test/data/test_standard.fa .
```

## 2) Start running the program
Then let's start running the program. Step 1 called `iterator` has a lot of inputs, but that is just because I wanted to give users a lot of flexibility on thermodynamic parameters for primer3.

Let's display the help message and explain the parameters:

```
primer_tk iterator -h

usage: primer_tk iterator [-h] -ref REF_GENOME -in REGIONS_FILE
                          [-opt_size PRIMER_OPT_SIZE]
                          [-min_size PRIMER_MIN_SIZE]
                          [-max_size PRIMER_MAX_SIZE] [-opt_gc PRIMER_OPT_GC]
                          [-min_gc PRIMER_MIN_GC] [-max_gc PRIMER_MAX_GC]
                          [-opt_tm PRIMER_OPT_TM] [-min_tm PRIMER_MIN_TM]
                          [-max_tm PRIMER_MAX_TM] [-sr PRODUCT_SIZE_RANGE]
                          [-flank FLANKING_REGION_SIZE] [-st SEQUENCE_TARGET]
                          [-mp MISPRIMING] [-tp THERMOPATH]

optional arguments:
  -h, --help            show this help message and exit
  -ref REF_GENOME, --ref_genome REF_GENOME
                        Reference Genome File to design primers around
  -in REGIONS_FILE, --regions_file REGIONS_FILE
                        File with regions to design primers around
  -opt_size PRIMER_OPT_SIZE, --primer_opt_size PRIMER_OPT_SIZE
                        The optimum primer size for output, default: 22
  -min_size PRIMER_MIN_SIZE, --primer_min_size PRIMER_MIN_SIZE
                        The optimum primer size for output, default: 18
  -max_size PRIMER_MAX_SIZE, --primer_max_size PRIMER_MAX_SIZE
                        The optimum primer size for output, default: 25
  -opt_gc PRIMER_OPT_GC, --primer_opt_gc PRIMER_OPT_GC
                        Optimum primer GC, default: 50
  -min_gc PRIMER_MIN_GC, --primer_min_gc PRIMER_MIN_GC
                        Minimum primer GC, default: 20
  -max_gc PRIMER_MAX_GC, --primer_max_gc PRIMER_MAX_GC
                        Maximum primer GC, default: 80
  -opt_tm PRIMER_OPT_TM, --primer_opt_tm PRIMER_OPT_TM
                        Optimum primer TM, default: 60
  -min_tm PRIMER_MIN_TM, --primer_min_tm PRIMER_MIN_TM
                        minimum primer TM, default: 57
  -max_tm PRIMER_MAX_TM, --primer_max_tm PRIMER_MAX_TM
                        maximum primer TM, default: 63
  -sr PRODUCT_SIZE_RANGE, --product_size_range PRODUCT_SIZE_RANGE
                        Size Range for PCR Product, default=200-400
  -flank FLANKING_REGION_SIZE, --flanking_region_size FLANKING_REGION_SIZE
                        This value will select how many bases up and
                        downstream to count when flanking SNP (will do 200 up
                        and 200 down), default: 200
  -st SEQUENCE_TARGET, --sequence_target SEQUENCE_TARGET
                        default: 199,1, should be half of your flanking region
                        size, so SNP/V will be included.
  -mp MISPRIMING, --mispriming MISPRIMING
                        full path to mispriming library for primer3 (EX:
                        /home/dkennetz/mispriming/humrep.ref
  -tp THERMOPATH, --thermopath THERMOPATH
                        full path to thermo parameters for primer3 to use (EX:
                        /home/dkennetz/primer3/src/primer3_config/) install
                        loc
```

As you can see, the help message is pretty detailed. The only thing that may need further explaining is the -mp (mispriming flag). This is just a file containing human repetitive elements or sequences that are common primer mispriming locations. Primer3 will use this to score primers in that location with a penalty so they are not likely to be selected.

Anything with a default value does not have to be specified on the command line (if you want to use that value as the parameter). These have been selected because I have seen good success with these values for standard pcr. In this tutorial, I will specify every value, but again you do not have to specify defaults.

*NOTE: THE -mp AND -tp FLAGS SHOULD BE THE FULL PATH TO FILE AS THIS IS WHAT PRIMER3 REQUIRES*
```
primer_tk iterator \
> -ref test_standard.fa \
> -in input_standard.csv \
> -opt_size 22 \
> -min_size 18 \
> -max_size 25 \
> -opt_gc 50 \
> -min_gc 20 \
> -max_gc 80 \
> -opt_tm 60 \
> -min_tm 57 \
> -max_tm 63 \
> -sr 200-400 \
> -flank 200 \
> -st 199,1 \
> -mp /home/dkennetz/tmp/tutorial/humrep.ref \
> -tp /home/dkennetz/primer3/src/primer3_config/
```

This module is just used to parse any reference genome and setup a primer3 config. The outputs will be:

```
flanking_regions.input_standard.fasta
primer3_input.input_standard.txt
```
The fasta will show us what sequences we pulled down, and the primer3_input will be the input to primer3.
The output files will maintain the same name as the input file used for the `-in` flag.

## 3) Run Primer3

This step is easy enough but can be time consuming for large datasets. In our case, it should take about 30 seconds.

```
primer3_core --output=primer_dump.txt primer3_input.input_standard.txt
```

This will output a log file named primer_dump.txt (we want) and a bunch of intermediate files (we don't want).
I like to clean the intermediates up (the intermediates also aren't kept if you use CWL).

```
rm -rf *.int *.for *.rev
```

## 4) Run the pre-pcr step and the multiplex filtering

This step is used for parsing the primer3 output file and setting up the input for pcr, and more importantly for filtering for multiplex primer pools. Let's look at the help message:

```
primer_tk pre -h
usage: primer_tk pre [-h] -d DUMP -o OUTFILE [-nd NO_DIMER]
                     [-spcr STANDARD_PCR_FILE] [-mpcr MULTIPLEX_PCR_FILE]
                     [-pa PERCENT_ALIGNMENT] -pcr {standard,multiplex}

Command Line argument for total primerinput file to check if primers have a
degreeof complementarity with each other as definedby the user. Default is 60%
(fairly strict).

optional arguments:
  -h, --help            show this help message and exit
  -d DUMP, --primer3_dump DUMP
                        Primer3 stdout passed into a 'dump' file to be used as
                        input
  -o OUTFILE, --outfile_name OUTFILE
                        The output filename for all primer information.
  -nd NO_DIMER, --no_dimer NO_DIMER
                        The primers left after dimers removed.
  -spcr STANDARD_PCR_FILE, --standard_pcr_file STANDARD_PCR_FILE
                        The file to be used for standard pcr input
  -mpcr MULTIPLEX_PCR_FILE, --multiplex_pcr_file MULTIPLEX_PCR_FILE
                        The file to be used for multiplex pcr input
  -pa PERCENT_ALIGNMENT, --percent_alignment PERCENT_ALIGNMENT
                        Percent match between 2 primers for pair to be
                        discarded. EX: primer_len = 22, percent_aln = 60
                        dimer_len = (60/100) * 22 = 13.2 -> 13.
  -pcr {standard,multiplex}, --pcr_type {standard,multiplex}
                        perform standard or multiplex pcr on given inputs.
```

### For multiplex PCR:

PrimerTK does two important things for multiplexing:
1. It checks for the presence of primer dimers between any two primers in the entire dataset.
2. It sets up an all vs all PCR input.

Both of these are important because in multiplexing, all primers will be in a single pool, so they will interact with each other. If the reaction between each other is more favorable than the reaction with the DNA template, or it is competing, those primers will have less effective concentration because they will bind to each other instead of the template. This will give us lower or no representation of the expected site.

Furthermore, we do in silico PCR to check for potential off-target amplification. This has the same effect as primer dimerization to a large extent. If there are competing sites, there is less effective concentration of primer at the site of interest. 

To deal with this, a simple string matching algorithm has been implemented. This returns a score based on how well the primers align with each other. I also ask for a user input of the `-pa` or percent alignment. I use this percent alignment strategy because primers will be different lengths. If the primer length times the percent alignment is greater than the calculated alignment score, then the primer is dropped. 

This is repeated until every possible combination in the pool has been checked.

Afterwards, it returns a primer file and a pcr file that only contains the post-filter primer pairs.

to run:

```
primer_tk pre \
 -d primer_dump.txt \
 -o total_primers.csv \
 -nd no_dimers.csv \
 -mpcr multiplex_pcr.txt \
 -pa 60 \
 -pcr multiplex
```

We can see from our example if we check the `no_dimers.csv` file, that with this pa score the primer pair was not filtered. However, as the size of the dataset increased, there is much more potential for primer-primer interaction and many more will be filtered. 


