# Study gene expression using RNA-seq analysis on Quest
*Thanks to Yoonie Joo for helping to create these workshop materials with Research Computing for the Genomics Community at Northwestern.  Please send any questions or comments to janna.nugent@northwestern.edu.*

- [Introduction to RNA-seq](#introduction-to-RNA-seq)
- [Set up your environment on the Genomic Compute Cluster on Quest](#set-up-your-environment-on-the-genomic-compute-cluster-on-quest)
- [Quality Check - Alignment - Estimate transcript abundances of RNA-seq reads](#quality-check---alignment---estimate-transcript-abundances-of-rna-seq-reads)
- [Expression and Differential Expression analysis](#expression-and-differential-expression-analysis)
- [Visualization of the results](#visualization-of-the-results)

___

## Introduction to RNA-seq 
*Note: in addition to the software taught in this workshop Northwestern researchers have access to CETO, an RNA-seq pipeline on Quest developed by Dr. Elizabeth Bartom.  Information on CETO is available [here](https://github.com/ebartom/NGSbartom); to use CETO on Quest, please contact e-bartom@northwestern.edu.*

There are several sequencing analysis methods available based on the **goal of your experiment**:
*	_**RNA-seq**_ : _co-expression networks, differentially expressed genes_ 
*	_**Chip-Seq | ATAC-seq | DNase-seq**_ : _Gene regulation dynamics, chromatin modeling_ 
*	_**Whole-genome Seq**_ : _Rare variant analysis_ 

**RNA-seq** could answer the following experimental questions:
*	_Measure expression variation within or between species_
*	_Transcriptome characterization_ 
*	_Identify splicing sites_
*	_Discover novel transcript_
*	_Differential expression studies of a gene in different conditions, etc._

Before you start, you have to consider the **experiment design**:
*	_What resources do you have already? (reference genome, curated genes, etc)_
*	_Do you need biological replications? (usually yes)_
*	_Do you need technical replications? (mostly not)_
*	_Do you need controls?_
*	_Do you need deep sequencing coverage?_

**Types of RNA-seq reads**

| - | Notes |
| :------------ | :-------------|
| Single-end |* Fast run <br> * less expensive | 
| Paired-end |* More data for each fragment/alignment/assembly <br> * Good for isoform-detection <br> * Good for detecting structural variations |
              
* Sanger sequencing - fasta format (1 header followed by any number of sequences lines) 
* NGS sequencing - fastq (Repeated 4 lines) 
    * Note that there are two fastQ files per sample in paired-end sequencing (+ strand, - strand) 
    * forward/reverse reads have almost same headers. 


___
## Set up your environment on the Genomic Compute Cluster on Quest

If you would like to perform RNA-seq on Quest, you need to first do the followings:

1. If you don't already have one, [apply for an account on Quest](https://www.it.northwestern.edu/secure/forms/research/allocation-request.html)
2. [Request access to the Genomic Compute Cluster on Quest](https://kb.northwestern.edu/page.php?id=78602) 
3. [Log in to Quest](https://kb.northwestern.edu/page.php?id=70706)
4. In this protocol, we will run an example analysis with chromosome X data of Homo sapiens. (Ref: [Nature Protocol 2016](https://www.nature.com/articles/nprot.2016.095)) 
    - All necessary data you need are available in the following directory on Quest: /projects/genomicsshare/RNAseq_workshop
      -	`'samples'` directory contains paired-end RNA-seq reads for 6 samples, 3 male and 3 female subjects from YRI (Yoruba from Ibadan, Nigeria) population. 
      -	`‘indexes’` directory contains the indexes for chromosome X for HISAT2. 
      -	`‘genome’` directory contains the sequence of human chromosome X (GrCH38 build 81)
      -	`‘genes’` directory contains human gene annotations for GrCH38 from RefSeq database. 
      -	`‘mergelist.txt’` and ‘geuvadis_phenodata.csv’ are exemplary scripts that you might want to write yourself in a text editor. 
      -	Since it is paired-end reads, each sample has two files: all sequence is in compressed 'fastq' format
        -	(cf) Our analysis only contains the genome of chromosome X, but if someone is interested in the full data sets, these files are ~25 times larger and you can find them [here](ftp://ftp.ccb.jhu.edu/pub/RNAseq_protocol)
5. Move to your home directory and copy the workshop directory into it: 
```bash
	cd ~ 		 
	cp -R /projects/genomicsshare/RNAseq_workshop .	
```

#### Software setup/installation:
6. Load the necessary modules on Quest: fastqc, samtools, HISAT2
```bash
	module load fastqc/0.11.5
	module load hisat2/2.0.4  
	module load samtools/1.6 
	module load stringtie/1.3.4 
```


___
## Quality Check - Alignment - Estimate transcript abundances of RNA-seq reads

#### Step1. Analyze raw reads’ quality with [FastQC](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/)
###### Input: bam/sam/fastQ file	:heavy_minus_sign:	Output: zip/html file contains quality report for each read
-	We have to confirm average quality per read, consistency, GC content (PCR bias), adapter/k-mer content, excessive duplicated reads, etc.
-	You can specify the list of adapter sequences with –adapters (-a) specifier. 
-	HTML ouput generated by FastQC helps visual inspection of overall read quality. Not all yellow and red highlights are problematic, so look through the reports with a grain of salt. 
-	*Other software options: __FastQC__ is for illumina read, __NGSQC__ is for basically every platform*

```bash
* You can get software instructions by typing a commandline: fastqc -h 
	
	mkdir qualitycheck
	fastqc --outdir ./qualitycheck/ ./samples/*_chrX_*.fastq.gz
```
One of the output files for FastQC can be viewed [here](http://htmlpreview.github.com/?https://github.com/nuitrcs/rnqseq_workshop/blob/master/ERR188234_chrX_1_fastqc.html).
#### Step2. Filter raw reads with [Trimmomatic](http://www.usadellab.org/cms/index.php?page=trimmomatic)
###### Input: fastQ file before filtering	:heavy_minus_sign:	Output: fastQ file after filtering
-	Even if our data looks fine, it is always a good idea to filter out low/poor quality reads. 
-	Appropriate threshold should be determined by each experiment design or organism. 
-	Before running the filtering step, you need to specify sequencing method of your reads since the software commands are different. (single-ended or paired-ended?)
	-	Our data are paired-ended, so we use ‘PE’ command for Trimmomatic.
	-	Trimmomatric removes adapter sequences, low quality reads, too-short reads, etc.
	-	Options you can include in Trimmomatric commands: 
	
		- a. ILLUMINACLIP: Cut adapter and other illumina-specific sequences from the read.
		- b. SLIDINGWINDOW: Perform a sliding window trimming, cutting once the average quality within the window falls below a threshold.
		- c. LEADING: Cut bases off the start of a read, if below a threshold quality
		- d. TRAILING: Cut bases off the end of a read, if below a threshold quality
		- e. CROP: Cut the read to a specified length
		- f. HEADCROP: Cut the specified number of bases from the start of the read
		- g. MINLEN: Drop the read if it is below a specified length
		- h. TOPHRED33: Convert quality scores to Phred-33
		- i. TOPHRED64: Convert quality scores to Phred-64

	-	It works with FASTQ (using phred + 33 or phred + 64 quality scores, depending on the Illumina pipeline used), either uncompressed or gzipp'ed FASTQ. Use of gzip format is determined based on the .gz extension.

-	*Another software option: [Fastx-toolkit](http://hannonlab.cshl.edu/fastx_toolkit/commandline.html)__ (available on Quest)*
	-	Optional instruction - FastX-toolkit does not accept gzip compressed files, so we would better make pipe and output in compressed format. The following command allows us to throw out any read that fails to meet a threshold of at least 70% of bases with Phred quality score > 20.
		- Load FastQC on Quest: `module load fastx_toolkit/0.0.14` 
		- Filtering command: `gunzip -c ./samples/ERR188273_chrX_1.fastq.gz | fastq_quality_filter -q 20 -p 70 -i -z -o ERR188273_chrX_1_fastqqc_filtered.fastq` 

```bash
* Paired ended:
	java -jar trimmomatic-0.33.jar PE -threads 1 -phred33 ./samples/ERR188273_chrX_1.fastq.gz ./samples/ERR188273_chrX_2.fastq.gz ./ERR188273_chrX_1_paired_filtered.fastq.gz ./ERR188273_chrX_1_unpaired_filtered.fastq.gz ./ERR188273_chrX_2_paired_filtered.fastq.gz ./ERR188273_chrX_2_unpaired_filtered.fastq.gz LEADING:3 TRAILING:3 SLIDINGWINDOW:70:20 MINLEN:30 
	java -jar trimmomatic-0.33.jar PE -threads 1 -phred33 ./samples/ERR188044_chrX_1.fastq.gz ./samples/ERR188044_chrX_2.fastq.gz ./ERR188044_chrX_1_paired_filtered.fastq.gz ./ERR188044_chrX_1_unpaired_filtered.fastq.gz ./ERR188044_chrX_2_paired_filtered.fastq.gz ./ERR188044_chrX_2_unpaired_filtered.fastq.gz LEADING:3 TRAILING:3 SLIDINGWINDOW:70:20 MINLEN:30 
	java -jar trimmomatic-0.33.jar PE -threads 1 -phred33 ./samples/ERR188104_chrX_1.fastq.gz ./samples/ERR188104_chrX_2.fastq.gz ./ERR188104_chrX_1_paired_filtered.fastq.gz ./ERR188104_chrX_1_unpaired_filtered.fastq.gz ./ERR188104_chrX_2_paired_filtered.fastq.gz ./ERR188104_chrX_2_unpaired_filtered.fastq.gz LEADING:3 TRAILING:3 SLIDINGWINDOW:70:20 MINLEN:30 
	java -jar trimmomatic-0.33.jar PE -threads 1 -phred33 ./samples/ERR188234_chrX_1.fastq.gz ./samples/ERR188234_chrX_2.fastq.gz ./ERR188234_chrX_1_paired_filtered.fastq.gz ./ERR188234_chrX_1_unpaired_filtered.fastq.gz ./ERR188234_chrX_2_paired_filtered.fastq.gz ./ERR188234_chrX_2_unpaired_filtered.fastq.gz LEADING:3 TRAILING:3 SLIDINGWINDOW:70:20 MINLEN:30 
	java -jar trimmomatic-0.33.jar PE -threads 1 -phred33 ./samples/ERR188454_chrX_1.fastq.gz ./samples/ERR188454_chrX_2.fastq.gz ./ERR188454_chrX_1_paired_filtered.fastq.gz ./ERR188454_chrX_1_unpaired_filtered.fastq.gz ./ERR188454_chrX_2_paired_filtered.fastq.gz ./ERR188454_chrX_2_unpaired_filtered.fastq.gz LEADING:3 TRAILING:3 SLIDINGWINDOW:70:20 MINLEN:30 
	java -jar trimmomatic-0.33.jar PE -threads 1 -phred33 ./samples/ERR204916_chrX_1.fastq.gz ./samples/ERR204916_chrX_2.fastq.gz ./ERR204916_chrX_1_paired_filtered.fastq.gz ./ERR204916_chrX_1_unpaired_filtered.fastq.gz ./ERR204916_chrX_2_paired_filtered.fastq.gz ./ERR204916_chrX_2_unpaired_filtered.fastq.gz LEADING:3 TRAILING:3 SLIDINGWINDOW:70:20 MINLEN:30 

This will perform the following:
•	Remove leading low quality or N bases (below quality 3) (LEADING:15)
•	Remove trailing low quality or N bases (below quality 3) (TRAILING:15)
•	Scan the read with a 70-base wide sliding window, cutting when the average quality per base drops below 20 (SLIDINGWINDOW:70:20)
•	Drop reads below the 30 bases long (MINLEN:30)
•	Convert quality score to Phred-33 
```

#### Step3. Re-analyze the quality of filtered reads with FastQC
###### Input: filtered fastQ file	:heavy_minus_sign:	Output: zip/html file contains quality report for each read

-	We have to confirm the read quality after filtering. 
-	You can determine this easily by re-running FastQC on the output fastq files. 
- 	We did not remove many, so this step could be optional. 
```bash
	fastqc --outdir ./qualitycheck/filtered/ *_chrX_*_filtered.fastq.gz
```

#### Step4. Alignment of RNA-seq reads to the genome with [HISAT](https://ccb.jhu.edu/software/hisat2/index.shtml)
###### Input: FastQ reads (2 per sample)	:heavy_minus_sign:	Output: SAM files (1 per sample)
-	HISAT2 (v 2.1.0) maps the reads for each sample to the ref genome: [Help page: `hisat2 –h` ]
-	Note that HISAT2 commands for paired(-1,-2) /unpaired (-U) reads are different. 
-	HISAT2 also provides an option, called --sra-acc, to directly work with NCBI Sequence Read Archive (SRA) data over the internet. This eliminates the need to manually download SRA reads and convert them into fasta/fastq format, without much affecting the run time. 
	-	(e.g.) `--sra-acc SRR353653,SRR353654`

```bash
	hisat2 -p 1 --dta -x ./indexes/chrX_tran -1 ./ERR188044_chrX_1_paired_filtered.fastq.gz -2 ./ERR188044_chrX_2_paired_filtered.fastq.gz -S ERR188044_chrX.sam
	hisat2 -p 1 --dta -x ./indexes/chrX_tran -1 ./ERR188104_chrX_1_paired_filtered.fastq.gz -2 ./ERR188104_chrX_2_paired_filtered.fastq.gz -S ERR188104_chrX.sam
	hisat2 -p 1 --dta -x ./indexes/chrX_tran -1 ./ERR188234_chrX_1_paired_filtered.fastq.gz -2 ./ERR188234_chrX_2_paired_filtered.fastq.gz -S ERR188234_chrX.sam
	hisat2 -p 1 --dta -x ./indexes/chrX_tran -1 ./ERR188273_chrX_1_paired_filtered.fastq.gz -2 ./ERR188273_chrX_2_paired_filtered.fastq.gz -S ERR188273_chrX.sam
	hisat2 -p 1 --dta -x ./indexes/chrX_tran -1 ./ERR188454_chrX_1_paired_filtered.fastq.gz -2 ./ERR188454_chrX_2_paired_filtered.fastq.gz -S ERR188454_chrX.sam
	hisat2 -p 1 --dta -x ./indexes/chrX_tran -1 ./ERR204916_chrX_1_paired_filtered.fastq.gz -2 ./ERR204916_chrX_2_paired_filtered.fastq.gz -S ERR204916_chrX.sam

* Confirm the QC parameters  
```

-	Parameter for QC: proportion of mapped read on either genome/transcriptome 
	-	To confirm sequencing accuracy and contaminated DNA 
		- If RNA-seq reads are mapped to _human genome_ – 70~90% + a few multi-mapping reads 
		- If RNA-seq reads are mapped to _transcriptome_ – less mapping % + more multi-mapping reads by sharing same exon among isoforms 
	- If the result screen says that some reads aligned discordantly, it means some occurrences of infusion or translocation. Possibly mismatched/too-far paired-end reads. 
-	_Other software options: **Picard, STAR, PSeQC, Qualimap**_




#### Step 5. Sort and convert the SAM file to BAM with [samtools](https://github.com/samtools/samtools)
###### Input: SAM file	:heavy_minus_sign:	Output: BAM file 
-	Samtools (v 2.1.0) sorts and converts the SAM file to BAM: [Help page: `samtools –help`]
-	Both SAM/BAM formats represent alignments. BAM is more compressed format. Unmapped reads may also be in the BAM file. Reads that map to multiple location will show up multiple times as well.
-	Exceeding mapping percentage over 100% does not indicate how many reads mapped. They can be inflated by filtering low q reads prior to alignment.

```bash
	samtools sort -@ 1 -o ERR188044_chrX.bam ERR188044_chrX.sam
	samtools sort -@ 1 -o ERR188104_chrX.bam ERR188104_chrX.sam
	samtools sort -@ 1 -o ERR188234_chrX.bam ERR188234_chrX.sam
	samtools sort -@ 1 -o ERR188273_chrX.bam ERR188273_chrX.sam
	samtools sort -@ 1 -o ERR188454_chrX.bam ERR188454_chrX.sam
	samtools sort -@ 1 -o ERR204916_chrX.bam ERR204916_chrX.sam
```

#### Step 6. Assemble and quantify expressed genes and transcripts with [StringTie](https://ccb.jhu.edu/software/stringtie/)

- [ ]	(a) Stringtie assembles transcripts for each sample:
###### Input: BAM file + reference GTF file	:heavy_minus_sign:	Output: Assembled GTF file (1 per sample)
```bash
	stringtie -p 1 -G ./genes/chrX.gtf -o ERR188044_chrX.gtf -l ERR188044 ERR188044_chrX.bam
	stringtie -p 1 -G ./genes/chrX.gtf -o ERR188104_chrX.gtf -l ERR188104 ERR188104_chrX.bam
	stringtie -p 1 -G ./genes/chrX.gtf -o ERR188234_chrX.gtf -l ERR188234 ERR188234_chrX.bam
	stringtie -p 1 -G ./genes/chrX.gtf -o ERR188273_chrX.gtf -l ERR188273 ERR188273_chrX.bam
	stringtie -p 1 -G ./genes/chrX.gtf -o ERR188454_chrX.gtf -l ERR188454 ERR188454_chrX.bam
	stringtie -p 1 -G ./genes/chrX.gtf -o ERR204916_chrX.gtf -l ERR204916 ERR204916_chrX.bam
```

- [ ]	(b) Stringtie merges transcripts from all samples:
###### Input: multiple GTF files to be merged + mergelist.txt with filenames	:heavy_minus_sign:	Output: One merged GTF (will be used as a reference for relative comparisons among samples) 
```bash
	stringtie --merge -p 1 -G ./genes/chrX.gtf -o stringtie_merged.gtf mergelist.txt
```

- [ ]	(c) Stringtie estimates transcript abundances and create table counts for Ballgown (software used downstream):
###### Input: BAM file of each sample + one merged GTF	:heavy_minus_sign:	Output: several output files for Ballgown-analysis ready 
```bash
	stringtie -e -B -p 1 -G stringtie_merged.gtf -o ./ballgown/ERR188044/ERR188044_chrX.gtf ERR188044_chrX.bam
	stringtie -e -B -p 1 -G stringtie_merged.gtf -o ./ballgown/ERR188104/ERR188104_chrX.gtf ERR188104_chrX.bam
	stringtie -e -B -p 1 -G stringtie_merged.gtf -o ./ballgown/ERR188234/ERR188234_chrX.gtf ERR188234_chrX.bam
	stringtie -e -B -p 1 -G stringtie_merged.gtf -o ./ballgown/ERR188273/ERR188273_chrX.gtf ERR188273_chrX.bam
	stringtie -e -B -p 1 -G stringtie_merged.gtf -o ./ballgown/ERR188454/ERR188454_chrX.gtf ERR188454_chrX.bam
	stringtie -e -B -p 1 -G stringtie_merged.gtf -o ./ballgown/ERR204916/ERR204916_chrX.gtf ERR204916_chrX.bam
```
-	_Other software available: **HTSeq-count, featureCounts**_

___
## Expression and Differential Expression analysis

To quantify expression of transcript/genes among different conditions:
-	Count the number of mapped reads on each transcript  
-	Quantify gene-level expression with GTF (gene transfer format) files. 

###### Input: grouping info of the samples (csv file)	:heavy_minus_sign:	Output: SAM files (1 per sample)

#### Step 7. Run differential expression analysis with [Ballgown](https://bioconductor.org/packages/release/bioc/html/ballgown.html)

- [ ]	(a) R Environment setup (Load/install R packages: ballgown, RSkittleBrewer, genefilter, dplyr, devtools)
```R
	module load R
	R
	
	# R commands are indicated with '>' below:

	>library("devtools") 
	>source("http://www.bioconductor.org/biocLite.R")
	>biocLite(c("alyssafrazee/RSkittleBrewer","ballgown", "genefilter","dplyr","devtools"))
	
	>library("ballgown")
	>library("RSkittleBrewer") # for color setup
	>library("genefilter") # faster calculation of mean/variance
	>library("dplyr") # to sort/arrange results
	>library("devtools")  # reproducibility/installing packages
```

- [ ]	(b) Load phenotype data 
	- In your future experiment, create your own phenotype data specifying the sample conditions you would like to compare. 
	- Each sample information presents on each row of the file 
```R
	> pheno_data = read.csv("phenodata.csv")
	> head(pheno_data)
```

- [ ]	(c) Read in the expression data that were calculated by Stringtie in previous step 6-(c)
	-	IDs of the input files should be matched with the phenotype information. 
	-	Ballgown also supports Cufflinks/RSEM data
```R
	> chrX <- ballgown(dataDir="ballgown", samplePattern="ERR", pData=pheno_data)
	> str(chrX)
```

- [ ]	(d) Filter to remove low-abundance genes with a variance across samples less than one
	-	Genes often have very few or zero counts.
	-	We can apply a variance filter for gene expression analysis. 
```R
	> chrX_filtered <- subset(chrX, "rowVars(texpr(chrX)) >1", genomesubset=TRUE)
	> str(chrX_filtered)
```


#### Step 8. Identify transcripts/genes that show statistically significant differences between groups

- [ ]	(a) Identify **transcripts** that show statistically significant differences between groups 
```R
	* We will look for transcripts that are differentially expressed between sexes, while correcting for any differences in expression due to the condition variable. 

	> results_transcripts <- stattest(chrX_filtered, feature="transcript", covariate="sex", adjustvars=c("condition"), getFC=TRUE, meas="FPKM")
	> head(results_transcripts)

	* Add gene names and gene IDs to the results:

	> results_transcripts <- data.frame(geneNames=ballgown::geneNames(chrX_filtered), geneIDs=ballgown::geneIDs(chrX_filtered), results_transcripts)
	> head(results_transcripts)
```

- [ ] 	(b) Identify **genes** that show statistically significant differences between groups 
```R
	> results_genes <- stattest(chrX_filtered, feature="gene", covariate="sex", adjustvars=c("condition"), getFC=TRUE, meas="FPKM")
	> head(results_genes)
```


-	Several **measurement criteria for RNA-seq quantification**:
	-	**_RPKM (reads per kilobase of exon model per million reads)_** adjusts for feature length and library size by sample normalization 
	-	**_FPKM (fragment per kilobase of exon model per million mapped reads)_** adjusts sample normalization of transcript expression (= similar to RPK) 
		-	RPKM = FPKM (in Single-end sequencing) 
		-	_FPKM can be translated to TPM_
	-	**_TPM (transcript per million)_** is used for measuring RNA-seq gene expression by adjusting transcript differences with overall read # in library: useful in comparing inter-sample comparison with different origins/compositions 
		-	Gene length is not important for inter-sample gene expression comparison, but important in ranking intra-sample gene expression 
		-	All the measures above are not useful for the samples with transcript variances. 

-	In differential expression analysis, we have to eliminate systematic effects that are not due to biological causal differences of interest. _(Normalization)_ We should condition the non-biological differences such as _sequencing depth, gene’s length, and variability._ Therefore we must calculate the fraction of the reads for each gene compared to the total amount of reads and to the whole RNA library. 
-	**Tests for significance** must rely on assumptions about the underlying read distributions. The _**negative binomial distribution**_ is often used for modeling gene expression between biological replicates, because it better accounts for noise than the **_Poisson distribution_**, which would otherwise be applicable as an approximation of the **_binomial distribution_** (presence of individual reads or not) with a large n (read library) and a small np. Another common assumption is that the majority of the transcriptome is unchanged between the two conditions. If these assumptions are not met by the data, the results will likely be incorrect. This is **_why it’s important to examine and perform QC on the expression data before running a differential expression analysis!_**
-	It is highly recommended to have at least two replicates per group. 

-	We can compare the transcripts that are differentially expressed between groups, while correcting for any different expression due to _**‘confounding’**_ variable. 
	-	Ballgown can look at the confounder-adjusted fold change (FC) between the two groups by setting getFC=TRUE parameter in stattest() function. 

-	_**For small sample size (n<4 per group)**_, it is often better to perform _regularization_ than standard linear model-based comparison as Ballgown does. Like “limma-voom” package in Bioconductor, DESeq, edgeR for gene/exon counts are the mostly used ones. (not appropriate for FPKM abundance estimates) 

-	_Other software options: **HTseq-count, DESeq2, edgeR, Kallisto, RSEM** (use expectation maximation to measure TPM value), **NURD** (Transcript expression measure in SE reads with low memory and computing cost), **Cufflinks** (using Tophat mapper for mapping, expectation-maximization algorithm)_

#### Step 9. Explore the results! 
```R
	* Sort the results from the smallest-largest p-value
	
	> results_transcripts <- results_transcripts[order(results_transcripts$pval),]
	> results_genes <- results_genes[order(results_genes$pval),]

	* What are the top transcript/gene expressed differently between sexes? 

	> head(results_transcripts)
	> head(results_genes)

	* (cf) You can also try filtering with q-value (<0.05) with subset() function.
	* Save the analysis results to csv files:

	> write.csv(results_transcripts, file="DifferentialExpressionAnalysis_transcript_results.csv", row.names=FALSE)
	> write.csv(results_genes, file="DifferentialExpressionAnalysis_gene_results.csv", row.names=FALSE)
	> save.image()			# your workspace will be saved as '.RData' in current working directory

```


___
## Visualization of the results

#### Step 10. Choose your environment for Visualization
Not only for visualizing the expression differences, the step is also essential for checking additional quality control criteria such as PCR duplication caused by variant calling. In our examples, we will use _**R Ballgown package**_ for RNA-seq analysis specific visualization.

#### :heavy_check_mark: _For small and moderately sized interactive analysis:_ 
-	Go to Rstudio-Quest analytics node on your browser [[https://rstudio.questanalytics.northwestern.edu/auth-sign-in]](https://rstudio.questanalytics.northwestern.edu/auth-sign-in)
-	You might have to re-install the required R packages for differential data analysis described above
```R
	> setwd("/YourWorkingDirectory/")
	> load(".RData")
```

#### :heavy_check_mark: _For large sized interactice analysis that might require over 4GB of RAM or more than 4+ cores:_
-	Request an interactive session on a compute node [[link]](https://kb.northwestern.edu/69247)
```
	msub -I -l nodes=1:ppn=4 -l walltime=01:00:00 -q genomics -A b1042
```
	
> You can choose to use either __**IGV or UCSC Genome browser**__ for visualizing your overall outcome. 



#### Step 11. 

In our example script, we will explore:
-	**Boxplot to view the distribution of gene abundances across samples**:
	-	Variety of measurements can be compared and visualized other than FPKM values, such as splice junction, exon and gene in the dataset. 
	-	Log transformation is required sometimes to plot some FPKM data = 0. 
-	**Boxplot to view an individual expression of a certain transcript between groups**. 
-	Plot the structure/expression levels in a sample of all transcripts that share the same gene locus.
	-	We can plot their structure and expression levels by passing the gene name and the Ballgown object to the plotTranscripts function.
-	Plot average expression levels for all transcripts of a gene within different groups

- [ ]	(a). Plot for distribution of gene abundances across samples:
-	In this example, we compare the FPKM measurements for the transcripts colored by 'sex' variable in phenotype file. 
```R
	> fpkm <- texpr(chrX, meas='FPKM')
	> fpkm <- log2(fpkm +1)
	> boxplot(fpkm, col=as.numeric(pheno_data$sex), las=2,ylab='log2(FPKM+1)')
```

- [ ]	(b). Plot for individual expression of a certain transcript between groups: 
```R
	* Setup palette with your favorite colors

	> coloring <- c('darkgreen', 'skyblue', 'hotpink', 'orange', 'lightyellow')
	> palette(coloring)

	* Choose your transcript of interest

	* In this example, by looking head(results_transcripts), I choose to draw the 13th transcript in the dataset. (gene name "XIST") You can also decide the transcript/gene of your interest!
	> which(ballgown::geneNames(chrX)=="XIST")	# Find the row number of the interested transcript/gene in dataset: 1484 here 
	> ballgown::transcriptNames(chrX)[1484]		# get the transcript name in the gene of interest: NR_001564
	
	* Draw the expression plot 
	> plot(fpkm[1484,] ~ pheno_data$sex, border=c(1,2), main=paste(ballgown::geneNames(chrX)[1484], ' : ',ballgown::transcriptNames(chrX)[1484]), pch=19, xlab="sex", ylab='log2(FPKM+1)')
	> points(fpkm[1484,] ~ jitter(as.numeric(pheno_data$sex)), col=as.numeric(pheno_data$sex))

```
-	The output plot shows the name of the transcript (NR_001564) and the name of the gene (XIST) that contains it. 
	- 	[Question] _Can you tell the exclusive expression of XIST in females? (c.f. In females, the XIST gene is expressed exclusively from the inactive X chromosome, and it is essential for the initiation and spread of X inactivation, which is an early developmental process that transcriptionally silences one of the pair of X chromosomes)_

- [ ]	(c). Plot the structure/expression levels in a sample of all transcripts that share the same gene locus:
```R
	* We choose sample 'ERR204916' for plotting structure/expression level in the same genomic position. 
	
	> plotTranscripts(ballgown::geneIDs(chrX)[1484],chrX, main=c("Gene XIST in sample ERR204916"), sample=c("ERR204916"))
	
	* The output plot shows one transcript per row, colored by its FPKM level. 
```


- [ ]	(d). Plot the average expression levels for all transcripts of a gene within different groups:

	- Using plotMeans() function, specify which gene to plot and which variable to group by. 
```R
	> geneIDs(chrX)[1484]
		"MSTRG.495" 
	> plotMeans('MSTRG.495', chrX_filtered, groupvar="sex", legend=FALSE)
	> plotMeans(ballgown::geneIDs(bg_chrX)[1484], chrX, groupvar="sex", legend=FALSE)
```
###### Have fun playing! 


_Other Software Options_:
1.	_Read-level visualization software: **ReadXplorer, UCSC genome browser, integrative Genomics Viewer (IGV), Genome Maps, Savant**_ 
2.	_Gene expression analysis software: **DESeq2, DEXseq **_
3.	_**CummeRbound, Sashimi plot** (junction reads will be more intuitive and aesthetic), **SplicePlot** (can get sashimi, structure, hive plot for sQTL), **TraV** (integrates all the data analysis for visualization but cannot be used for huge genome)_


___

### Resources for Further Study 

- [RNA-seq wiki](https://github.com/griffithlab/rnaseq_tutorial/wiki)
- [RNA-seq analysis tutorial with Differential analysis in R DESeq2 package](https://github.com/CandiceChuDVM/RNA-Seq/wiki/RNA-Seq-analysis-tutorial)

##### Online courses
- [Bioconductor for Genomic Data Science (Coursera)](https://www.coursera.org/learn/bioconductor)
- [Command line tools for Genomic Data Science (Coursera)](https://www.coursera.org/learn/genomic-tools)
- [Genomic Data Analysis (edX)](https://www.edx.org/xseries/genomics-data-analysis)

##### Youtube lecture videos
- [Informatics for RNA-seq analysis (5 videos)](https://www.youtube.com/playlist?list=PL3izGL6oi0S849u7OZbX85WTyBxVdcpqx)
- [Introduction to RNA-sequencing (1hr 20mins)](https://youtu.be/Ji9nFCYl7Bk)
- [RPKM, FPKM and TPM](https://youtu.be/TTUrtCY2k-w)
