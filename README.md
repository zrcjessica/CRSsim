# CRSsim :sparkles:
This repository contains code and instructions for simulating data generated by CRISPR regulatory screens and assessing the performance of various analysis methods for analyzing CRISPR regulatory screens. It contains the following sections:

[Simulations](https://github.com/patfiaux/CRSsim/blob/master/README.md#1-simulating-crispr-regulatory-screen-data)

[Analysis and performance](https://github.com/patfiaux/CRSsim/blob/master/README.md#2-analyzing-simulated-data-and-evaluate-performance)

[Advanced flags](https://github.com/patfiaux/CRSsim/blob/master/README.md#3-advanced-flags)

# 1. Simulating data generated by CRISPR regulatory screens
This software can be used to simulate data that would be generated by a standard CRISPR regulatory screen.

The simulation parameters mimic the experimental procedure of a CRISPR regulatory screen by taking the following variables into account:

| Experimental step | Simulated step |
| ----------------- | -------------- |
| NA | Size and number of true enhancers|
| NA | Enhancer strength |
| NA | Guide efficiency |
| Transduction of cells with guide library | Generate guide distribution |
| Load sorter with cells* | Specify number of cells to sort |
| Sort cells** | Specify sorting probabilities for sorting using Dirichlet-Multinomial distribution |
| Specify sequencing depth | Specify sequencing depth |
| Sequence pools | Specify whether PCR duplicates are accounted for |

\* If a selection screen is specified, the user must provide the simulation with the number of cells in the input pool prior to sorting.

** dropout rate

## 1.1 Installations and setup for simulations
The simulations are executed in [R](https://cran.r-project.org/bin/windows/base/). Please make sure you have R version 3.5.1 or higher installed on your computer.

Clone the CRSsim repository to your computer with the following command: 
 ```bash
git clone https://github.com/patfiaux/CRSsim.git
```
or manually download the repository.

To simulate data, you will need the packages listed below. If you don't have them, install them using the following commands in R. Installations should take about 5 minutes on a standard laptop.
### 1.1.1 R packages
* MCMCpack
``` r
install.packages('MCMCpack')
```
* transport
``` r
install.packages('transport')
```
### 1.1.2 Bioconductor packages
``` r
if (!requireNamespace("BiocManager", quietly = TRUE))

    install.packages("BiocManager")
```
* IRanges
``` r 
BiocManager::install("IRanges", version = "3.8")
```
* GenomicRanges
``` r
BiocManager::install("GenomicRanges", version = "3.8")
```

## 1.2 Simulate a selection screen with sample data
Here is a walkthrough of simulating data for a selection screen with sample data provided in `./CRSsim/Example_data`.

Running a simulation will generate three .csv files:
* An **annotation file**, containing information about all the simulated guides (chromosome, start, end, label). The output file will be named `{output_name}_info.csv`.
* A **counts file**, containing the counts for each guide in each pool. The output file will be named `{output_name}_counts.csv`.
* An **enhancer file**, containing the locations of the simulated regulatory regions (e.g. enhancers). The output file will be named `{output_name}_enhancers.csv`.

Follow the steps below to simulate data for a selection screen. 

### 1.2.1. Source the script
We recommend that you navigate into the `./CRSsim/Example_simulations` directory (included in the repository) and generate all output files there. After navigating into that directory, run the following commands in R to access all functions called by the simulation script:
``` r
source('/path/to/script/CRSsim.r')
```

### 1.2.2. Setting up simulation flags
Running a simulation will generate an **annotation file**, a **counts file** and an **enhancer file** (described above).

We have created an empty directory for you (`./CRSsim/Example_simulations`), within which you can generate these output files. Navigate into this directory and begin setting the option flags. There are several different flags which have no default arguments and must be supplied by the user. Below is an outline on how to set these required flags.

1. Flags are set up as an R list object. Run the following command in R to create this object:
``` r
sim.flags <- list()
```
2. Set the output name for the simulation. All output files generated by this run of the simulation will contain this prefix in the filehandle. Here, we will name all outputs `Example_simulation`:
```r
sim.flags$simName <- 'Example_simulation'
```
3. Provide information about the intended guide targets. Either supply them directly, as is demonstrated here, or generate them within the script (see details under [Advanced Simulations](https://github.com/patfiaux/CRSsim/blob/master/README.md#31-advanced-simulations). The input should be a data frame object with columns for chromosome, start position and end position labeled `chrom`, `start`, and `end`, respectively. Each row represents a different guide and its target site location. For Cas9, CRISPRi, and CRISPRa screens, the distance between the start and end sites should be set to something small, such as $ \text{start} = \text{target site} - 20$ and $\text{end} = \text{target site}$. Here, we will supply the guide target information from `../Example_data/Example_selectionScreen_info.csv`:
```r
sim.flags$guides <- read.csv('../Example_data/Example_selectionScreen_info.csv', stringsAsFactors = F)
```
4. If guide targets are provided as in step 3, then the marker gene of interest should also be provided. This assumes that some of the guides provided are targeting the gene of interest and can serve as positive controls. The input should be a data frame object with the chromosome, start, and end sites of all exons of the gene of interest. The column names of this data frame should be `chrom`, `start`, and `end`. Here, we supply the exon information from `../Example_data/Example_gene.csv`.
```r
sim.flags$exon <- read.csv('../Example_data/Example_gene.csv', stringsAsFactors = F)
```
5. Provide the original guide distribution for each replicate. This can be supplied by an existing data set, as demonstrated below. Here, we  supply the guide distribution data from `../Example_data/Example_selectionScreen_counts.csv`. Another option is to generate the distributions using a zero-inflated negative binomial (ZINB) distribution. See [Advanced Simulations](https://github.com/patfiaux/CRSsim#31-advanced-simulations) for details about this option.
```r
example.counts <- read.csv('../Example_data/Example_selectionScreen_counts.csv', stringsAsFactors = F)
sim.flags$inputGuideDistr <- cbind(before_1 = example.counts$before_repl1, 
  before_2 = example.counts$before_repl2)
```  
6. Specify the screen type. The user can specify one of two screen types: either a `selectionScreen` as shown here  or a `FACSscreen`, where cells are sorted into different pools. See [FACS screen simulation](https://github.com/patfiaux/CRSsim#123-facs-screen-simulation-quickstart-with-example-data) for an example of simulating a FACS screen.
```r
sim.flags$selectionScreen <- TRUE
```
7. Provide names for the different pools in each replicate. In this case, each replicate will have a *before* and an *after* selection pool. Their names will be: `before_repl1`, `after_repl1`, `before_repl2`, etc. 
```r
sim.flags$poolNames <- c('before', 'after')
```
8. Specify the CRISPR system used to generate the data. For CRISPRi, the default range of the perturbation effect is assumed to be 1kb. However, this can be manually specified. See the [Advanced Simulations](https://github.com/patfiaux/CRSsim#31-advanced-simulations) section for more details on how to simulate data generated by other CRISPR systems.
```r
sim.flags$crisprSystem <- 'CRISPRi'
```
9. Specify the number and the size of enhancers to be simulated. They will be placed at random genomic positions throughout the screen.
```r
sim.flags$nrEnhancers <- 25
sim.flags$enhancerSize <- 50  # base pairs
```
10. Specify the sequencing depth for each of the pools. Here, the parameters have been set such that the average guide count is 15. A sequencing depth for each pool in each replicate must be defined:
```r
repl1.seqDepth <- c(nrow(sim.flags$guides) * 15, nrow(sim.flags$guides) * 15) # set depths for 'before' and 'after' pools of replicate 1
repl2.seqDepth <- c(nrow(sim.flags$guides) * 15, nrow(sim.flags$guides) * 15) # set depths for 'before' and 'after' pools of replicate 1
```
To set the `seqDepth` flag, you must provide a list object where each named list entry represents a replicate and its corresponding sequencing depth(s). 
```r
sim.flags$seqDepth <- list(repl1 = repl1.seqDepth, repl2 = repl2.seqDepth)
```
11. Additionally, the simulations can simulate data sets where PCR duplicates are either accounted for or not. If you would like to generate a data set where duplicates are accounted for, set the `pcrDupl` flag to `FALSE` or `F`. 

12. You must also specify:
* selection strength: how strong the effect of disrupting the gene of interest is (`high` or `low`).     
* guide efficiency: what proportion of the guides successfully perturb their targets (`high`, `medium`, or `low`)
* enhancer strength: how should the enhancer signal is (`high`, `medium`, or `low`)

Each of these parameters can be manually provided as strings. See [Advanced Simulations](https://github.com/patfiaux/CRSsim#31-advanced-simulations) for details.
```r
sim.flags$selectionStrength <- 'high'
sim.flags$guideEfficiency <- 'medium'
sim.flags$enhancerStrength <- 'medium'
```

13. Run the simulation! :sparkles:
```r
simulate_data(sim.flags)
```

## 1.2.3 Simulate a FACS screen with sample data
Here is a demonstration for simulating data generated by a FACS screen. Highlighted below are the main option differences from a selection screen simulation. 

1. Specify the type of screen as `FACSscreen` in your argument flags.
```r
sim.flags$selectionScreen <- FALSE
```
2. As in the selection screen example, each pool in each replicate must be named. The difference in this example is that there must be more than one pool. Here is an example where cells are sorted from an input pool into either a high, medium, or low expression pool.
```r
sim.flags$poolNames <- c('input', 'high', 'medium', 'low')
```
3. The sequencing depth must also be specified for each pool in each replicate. 
```r
sim.flags$seqDepth <- list(repl1 = rep(18e6, 4), repl2 = rep(18e6, 4) )
```

## Keep in mind!
An average guide count of 15 vs. 100 vs. 500 has a major effect on power to detect true signal when everything else is held constant. Make sure to adjust `seqDepth` when changing the number of guides used for simulating the data.

# 2. Analyze simulated data and evaluate performance
After simulating CRISPR screen data, you are ready to analyze it! CRSsim provides a script for analyzing the data using a number of methods and comparing their performance.

Running the analysis will generate the following outputs:
* a per-guide scores file, written to `{output_name}_guideScores.csv`
* a per-genome scores file, written to `{output_name}_genomeScores.csv`
* a bedGraph file for genome scores, written to `{output_name}_genomeScores.bedgraph`
* an AUC plot summarizing the performance of all methods compared, written to `{output_name}_perElement_AUCeval.pdf`
* a prAUC plot summarizing the performance of all methods compared, written to `{output_name}_perElement_prAUCeval.pdf`
* a .csv file summarizing the AUC and prAUC scores for each method evaluated, written to `{output_name}_perElement_method_eval.csv`

## 2.1 Installations and setup for analysis and performance evaluation

To analyze data and evaluate method performance you will need the R packages listed below. If you don't have them, install them using the following commands:
### 2.1.1 R packages
* dplyr
```r
install.packages('dplyr')
```
* ggplot2
```r
install.packages('ggplot2')
```
* pROC
```r
install.packages('pROC')
```
* glmmTMB
```r
install.packages('glmmTMB')
```
* extraDistr
```r
install.packages('extraDistr')
```
* MESS
```r
install.packages('MESS')
```

### 2.1.2 Bioconductor packages
```r
if (!requireNamespace("BiocManager", quietly = TRUE))

    install.packages("BiocManager")
```

* IRanges
```r
BiocManager::install("IRanges", version = "3.8")
```

* GenomicRanges
```r
BiocManager::install("GenomicRanges", version = "3.8")
```

* edgeR
```r
BiocManager::install("edgeR")
```

* DESeq2
```r
BiocManager::install("DESeq2")
```

## 2.2 Analyze sample selection screen data
We have provided sample selection screen data for analysis. This does not require the user to first simulate their own selection screen data; however, the example in [1.2](https://github.com/patfiaux/CRSsim#12-simulation-quickstart-with-example-data-selection-screen) walks through how to do this yourself.

For this example, we recommend navigating into the empty performance evaluation folder we have provided (`./CRSsim/Example_performanceEval`) and generating all output files there.

### 2.2.1 Source the script
After navigating into the analysis folder, open an R session and source the performance evaluation script by running the following command:
```r
source('/path/to/script/RELICS_performance.r')
```

### 2.2.2. Setting up analysis flags
1. Flags are set up as an R list object. Run the following command in R to create this object:
```r
analysis.specs <- list()
```

2. Set the output name for the analysis. All output files generated by the analysis will contain this prefix in the filehandle. Be sure to choose a different name from any existing output files.
```r
analysis.specs$dataName <- 'Example_performanceEval'
```
3. Specify the paths to the annotation file and the counts file generated in the simulation step. Here, we have provided example files so that the user does not have to run a full simulation prior to this example. Refer to [RELICS repo](https://github.com/patfiaux/RELICS) for information about file formats.
```r
analysis.specs$CountFileLoc <- '../Example_data/Example_simulation_counts.csv'
analysis.specs$sgRNAInfoFileLoc <- '../Example_data/Example_simulation_info.csv'
```
4. Multiple analysis methods can be applied to the data: RELICS, fold change, edgeR, and DESeq2. The user can specify which of these methods they would like to apply to the data:
```r
analysis.specs$Method <- c('RELICS-search', 'FoldChange', 'edgeR', 'DESeq2')
```
5. To run RELICS, you must clone the [RELICS GitHub](https://github.com/patfiaux/RELICS/) and source the RELICS script in the working directory **BEFORE** sourcing the performance script.
```r
source('/path/to/script/RELICS.r')
source('/path/to/script/RELICS_performance.r')
```
RELICS analysis instructions can be found [here](https://github.com/patfiaux/RELICS/blob/master/README.md#quickstart-with-example-data).
```r
analysis.specs$repl_groups <- '1,2;3,4'
analysis.specs$glmm_positiveTraining <- 'exon'
analysis.specs$glmm_negativeTraining <- 'neg' 
```
6. For edgeR, DESeq2, and fold change, select the pools to be compared against one another. Pools are referenced by their corresponding column index in the count file.
```r
analysis.specs$Group1 <- c(1,3)
analysis.specs$Group2 <- c(2,4)
```
7. For fold change, you need to specify whether the different pools are paired (from the same replicate with a 1-1 correspondence) or if there is an imbalance between the groups.
```r
analysis.specs$foldChangePaired <- 'yes' # else set to 'no'
```
8. Specify that results should be evaluated based on a set of regions known to be true positives and true negatives
```r
analysis.specs$simulated_data <- 'yes' # specify that the analysis is based on simulated data where the ground thruth is known
analysis.specs$pos_regions <- '../Example_data/Example_simulation_enhancers.csv' # file location of all known positive regions
analysis.specs$evaluate_perElement_Performance <- 'yes' # specify that the performance of different methods is to be evaluated
analysis.specs$positiveLabels <- 'pos' # label for regions which are true positives
analysis.specs$negativeLabels <- c('neg', 'chr') # labels for regions which are true negatives
```

9. Depending on the CRISPR system used, the range of perturbation effect will be different. We recommend setting the range to 20bp for `CRISPRcas9` and to 1000bp for `CRISPRi` and `CRISPRa`. Note that the effect range is added to the positions specified in the info file. If the effect range is already accounted for in the positions specified in the annotation file, then it should be set to zero here.

In case of a `dualCRISPR` system, an arbitrary `crisprEffectRange` can be specified as RELICS will automatically use the deletion range between guide 1 and guide 2 as effect range.
```r
analysis.specs$crisprSystem <- 'CRISPRi' # other options: CRISPRcas9, CRISPRa, dualCRISPR
analysis.specs$crisprEffectRange <- 1000
```

10. Once you have your flags set, create a specification file using the `write_specs_file()` function. The arguments and their names will be written to this file for future reference. The two arguments that this function takes are the list of flags you just set (`analysis.specs`) and the desired name of the output file (`.txt` will be automatically used as the file extension, so do not include any file extension for this argument).
```r
write_specs_file(analysis.specs, 'Example_performanceEval_specs')
```
11. Once the specification file has been set up, simply use the `analyze_data()` function to start the analysis. The example here should take about 5 minutes, depending on your operating system.
```r
analyze_data('Example_performanceEval_specs.txt')
```

# 3. Advanced Flags

## 3.1 Advanced Simulations

Guides and their targets can be simulated if not readily available. Both single-guide as well as dual-guide screens can be simulated. For both of these types, the number of guides (`nrGuides`) must be specified, as well as the screen type (`screenType`) and the step size between guides (`stepSize`). Additionally, if a dual CRISPR screen is selected, the deletion size must be specified (`stepSize`).  

Possible `screenType` options include: `CRISPRi`, `CRISPRa`, `Cas9` and `dualCRISPR`

If this option is chosen, all guides are abritrarily chosen to be located on chromosome 1 and ~5% of the guides will be selected to serve as positive controls.

```r
sim.flags$guides <- generate_guide_info(list(nrGuides = 10000, screenType = 'dualCRISPR', stepSize = 20, deletionSize = 1000))
```

The input count distribution for the different replicates can be taken from an existing data set. It is also possible to generalize existing distributions using the zero-inflated negative binomial distribution (ZINB). The ZINB has both a mean (rate) and a dispersion parameter, as well as a parameter describing the fraction of the distribution originating from the zero mass (eta). Below are the steps to obtain and use the parameters from a ZINB:
```r
# obtain ZINB parameters which descibe the distribution
before.repl1.par <- obtain_ZINB_pars(example.counts$before_repl1)
example.rate <- before.repl1.par$rate              # rate = 76.5
example.dispersion <- before.repl1.par$dispersion  # dispersion = 2.6
example.eta <- before.repl1.par$eta                # eta = 1e-4

# to generate a ZINB distribution with 15000 guides
before.repl1.simulated <- create_ZINB_shape(15000, example.eta, example.rate, example.dispersion)
before.repl2.simulated <- create_ZINB_shape(15000, example.eta, example.rate, example.dispersion)

# combine the two simulated replicates and set them as input distributions
sim.flags$inputGuideDistr <- cbind(before_1 = before.repl1.simulated, before_2 = before.repl2.simulated)
```  

Currently, four different CRISPR systems can be simulated: CRISPRi, CRISPRa, Cas9, and dualCRISPR.


By default, CRISPRi and CRISPRa are assumed to have an effect range of 1kb and Cas9 of 20bp. However, it is also possible to manually set this range with the `crisprEffectRange` flag.


For dualCRISPR, the effect range is equivalent to the deletion size. The deletion size introduced by two guides must be represented by 'start' set as the target site of guide 1 and 'end' as the target site of guide 2.
```r
# example for how to change the effect range of a CRISPR system used
sim.flags$crisprSystem <- 'CRISPRi'
sim.flags$crisprEffectRange <- 500
```

Both the guide efficiency and the enhancer strength are simulated from a beta distribution. The two parameters can be specified by setting `guideEfficiency` and `enhancerStrength` to either `high`, `medium` or `low`. It is also possible to directly specify the two shape parameters of the beta distribution. As a general rule, if the shape parameters provided are large, the observed distribution variance is reduced. The larger shape 1 parameter is compared to shape 2 parameter, the more the distribution will be skewed towards 1. To visualize this phenomenon, you can also plot the histogram by randomly sampling from a beta-distribution and then subsequently changing the shape parameters. The default parameters are:

    - high: enhancerShape1 = 7, enhancerShape2 = 2
    
    - medium: enhancerShape1 = 5, enhancerShape2 = 5
    
    - low: enhancerShape1 = 2, enhancerShape2 = 7
    
This also applies for `guideShape1` and `guideShape2`.

```r
hist(rbeta(10000, shape1 = 8, shape2 = 1))  # randomly generate 10000 instances of the beta distribution

sim.flags$guideEfficiency <- 'high'
sim.flags$enhancerStrenth <- 'high'

# the above is equivalent to what's below
sim.flags$enhancerShape1 <- 7
sim.flags$enhancerShape2 <- 2
sim.flags$guideShape1 <- 7
sim.flags$guideShape2 <- 2
```


Selection strength: To understand the details of the selection strength it is helpful to have some understanding of the Dirichlet distribution. Both for the selection screen as well as for the FACS screen the probability of each guide being either selected or sorted into a given pool is given by a Dirichlet random variate. As an example:


Cells are sorted into three pools: high, medium and low gene expression. The expected probability that a negative control will be sorted into each of the given pools are 0.48, 0.48, and 0.04, respectively. For each guide, this probability shifts slightly. The Dirichlet random variable provides this variation. All cells containing a negative control guide are subsequently assigned to each of the three pools with following probabilities:  `rdirichlet(1, c(48, 48, 4))`.

Continuing the example above: assume the expected probabilities that a positive control is sorted into each of the three pools - high, medium and low gene expression - are 0.45, 0.45, and 0.1, respectively. All cells containing a positive control guide is subsequently assigned to these pools by observing a Dirichlet random variable as follows:  `rdirichlet(1, c(45, 45, 10))`.

To manually set the Dirichlet probabilities, use the `posSortingFrequency` and the `negSortingFrequency` flags. To continue the example from above:
```r
sim.flags$posSortingFrequency <- c(45, 45, 10)
sim.flags$negSortingFrequency <- c(48, 48, 4)
```

Note: 

1. The sum of the frequencies does not have to equal 1. 

2. The larger the numbers chosen, the less variable the sorting becomes.


The defaults for the `high` flags were chosen based on their ability to accurately represent the sorting parameters for either a selection screen or a FACS screen. The default `low` flags were chosen as an arbitrary fraction of the `high` selection.

The default flags used for `high`:
```r
# for a selection screen:
sim.flags$posSortingFrequency <- c(1)
sim.flags$negSortingFrequency <- c(5)

# in a FACS screen sorted into 3 pools: 
# '97' is repeated for all pools except the last one
sim.flags$posSortingFrequency <- c(97, 97, 13) * 0.5
sim.flags$negSortingFrequency <- c(97, 97, 3) * 0.5
```

The default flags used for `low`:
```r
# for a selection screen:
sim.flags$posSortingFrequency <- c(4)
sim.flags$negSortingFrequency <- c(5)

# in a FACS screen sorted into 3 pools: 
# '97' is repeated for all pools except the last one
sim.flags$posSortingFrequency <- c(97, 97, 5) * 0.5
sim.flags$negSortingFrequency <- c(97, 97, 3) * 0.5
```

## 3.2 Advanced Analysis and performance evaluation
To analyze data with MAGeCK as used by [Diao et al. 2017](https://www.ncbi.nlm.nih.gov/pubmed/28417999), specify use of `alphaRRA` and the bins within which to tile the analyzed region. Note, `alphaRRA` will be applied to per-guide scores from all methods given by the `Method` flag.
```r
analysis.specs$postScoreAlphaRRA <- 'yes'
analysis.specs$binSize <- 50
```

To combine guide scores using a sliding window as in [Fulco et al. 2016](https://www.ncbi.nlm.nih.gov/pubmed/27708057) or [Simeonov et al. 2017](https://www.ncbi.nlm.nih.gov/pubmed/28854172), specify the number of guides to include per sliding window as well as the maximum window size (if the tiled deletion generates a 10KB gap it would not make sense to consider the entire region as one score).
```r
analysis.specs$postScoreSlidingWindow <- 'yes'
analysis.specs$guidePerSlidingWindow <- 15
analysis.specs$maxWindowSize <- 8000
```
