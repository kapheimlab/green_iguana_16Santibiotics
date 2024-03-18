## green iguana microbiome - antibiotics
## Lab study - Antibiotic knockdown of microbiome
## Karen M. Kapheim
## December 19, 2023

This is processing of 16S amplicon sequences in the Qiime2 pipeline.
Samples were collected with cloacal swabs from green iguanas in the Denardo lab 
at ASU.

DNA was extracted at USU, PCR and sequencing was performed at Shedd with
515f and 806rB primers.

PCR includes both a Zymobiomics Microbial Mock Community and a no template control.


NOTES: Working through tutorials on https://docs.qiime2.org/2018.8/interfaces/q2cli/
Working in interactive mode.

## Gather data

Sample data:

* Still waiting on a file that maps sequence ID to iguana ID and date of sampling.


Sequence data:

`/uufs/chpc.utah.edu/common/home/kapheim-group2/iguanas/green_iguana_antibiotics/FASTQ_Generation_2023`


Experimental Information: 

From Claudia Ki 2023nov03:

> We took baseline samples before the study began (day0) and then administered antibiotic treatments (clindamycin, penicillin, sterile water) for 3 days. After the 4th day, samples were collected again. Iguanas were either given a LPS or PBS injection (3X2 design). We took samples 24hr, 1 week, and 2 week after (termination).

Use all extraction blanks

Need to pull out the relevant samples.

#### Get sequences

For now, I am assuming we want to use everything except the `ZYMO*` sequences.

```
cd /uufs/chpc.utah.edu/common/home/kapheim-group2/iguanas/green_iguana_antibiotics
# Make a list of directories from which to copy seq files
#awk -F ',' '{print $19}' GreenIguanaManifest_subset.csv > #seq_paths.txt
# Remove the header line that was also printed
#sed '1d' seq_paths.txt > seq_paths_noheader.txt
# check
#wc -l GreenIguanaManifest_subset.csv
#wc -l seq_paths.txt
#wc -l seq_paths_noheader.txt
# 209 samples
# copy files
```


```
mkdir rawseqs
mkdir rawseq_dirs
cp -r ./FASTQ_Generation_2023/GI* ./rawseq_dirs/
cp -r ./FASTQ_Generation_2023/BG* ./rawseq_dirs/
cp -r ./FASTQ_Generation_2023/NTC* ./rawseq_dirs/
ls ./rawseq_dirs/ | wc -l
# 236 sequences - all but the ZYMOs

#cd rawseqs
#for FILE in $(cat ../seq_paths_copied.txt) ; do  cp -R ${FILE} ./; done

# get the sequences out of the directories
find ./rawseq_dirs -name '*.fastq' -exec mv {} ./rawseqs \;

# check that we have the correct number of seqs
ls ./rawseqs/ | wc -l
# 472

# delete the directory of directories
rm -R rawseq_dirs/
```

Now we should have just the fastq files we want in the `rawseqs` directory.

#### Make manifest

```
ls ./rawseqs/ > dietLPS_manifest.csv
```

Open in LibreOffice and add header and other columns

headers:
`sample-id,absolute-filepath,direction`

#### Set-up


```
salloc --time=24:00:00 --account=kapheim-np --partition=kapheim-np --nodes=1 -c 48 
#module load anaconda3/2019.03 qiime2/2019.4
module load anaconda3/2023.03 qiime2/2023.5
cd /uufs/chpc.utah.edu/common/home/kapheim-group2/iguanas/green_iguana_antibiotics
```

## Import sequence data

Working from https://docs.qiime2.org/2018.8/tutorials/importing/.


```
mkdir /uufs/chpc.utah.edu/common/home/kapheim-group2/iguanas/green_iguana_antibiotics/import
qiime tools import --type 'SampleData[PairedEndSequencesWithQuality]' \
  --input-path abX_manifest.csv \
  --output-path ./import/abx-pe-demux.qza \
  --input-format PairedEndFastqManifestPhred33
qiime demux summarize \
  --i-data ./import/abx-pe-demux.qza \
  --o-visualization ./import/abx-pe-demux.qzv
```


# Trim adapters

Following
https://forum.qiime2.org/t/demultiplexing-and-trimming-adapters-from-reads-with-q2-cutadapt/2313    
https://docs.qiime2.org/2018.8/plugins/available/cutadapt/trim-paired/    
https://rachaellappan.github.io/VL-QIIME2-analysis/pre-processing-of-sequence-reads.html   

Used primers for 515f and 806r.



```
mkdir /uufs/chpc.utah.edu/common/home/kapheim-group2/iguanas/green_iguana_antibiotics/trimmed
qiime cutadapt trim-paired \
  --i-demultiplexed-sequences ./import/abx-pe-demux.qza \
  --p-cores 48  \
  --p-front-f GTGYCAGCMGCCGCGGTAA \
  --p-front-r GGACTACNVGGGTWTCTAAT \
  --o-trimmed-sequences ./trimmed/abx-pe-trimmed.qza \
  --verbose
&> ./trimmed/primer_trimming.log
qiime demux summarize \
  --i-data ./trimmed/abx-pe-trimmed.qza \
  --o-visualization ./trimmed/abx-pe-trimmed.qzv
```

#### Check quality and sequence length

```
qiime tools view ./import/abx-pe-demux.qzv
qiime tools view ./trimmed/abx-pe-trimmed.qzv
```


## Denoising

Used visualization to choose length to truncate sequences to based on where median quality score
drops below 30.     

The reverse reads look a lot more variable in terms of quality. 

Optimal truncating:   

f-244, r-165



```
mkdir /uufs/chpc.utah.edu/common/home/kapheim-group2/iguanas/green_iguana_antibiotics/dada2
qiime dada2 denoise-paired \
  --p-n-threads 48 \
  --i-demultiplexed-seqs ./trimmed/abx-pe-trimmed.qza \
  --p-trim-left-f 0 \
  --p-trim-left-r 0 \
  --p-trunc-len-f 244 \
  --p-trunc-len-r 165 \
  --o-table ./dada2/abx_table.qza \
  --o-representative-sequences ./dada2/abx_repseqs.qza \
  --o-denoising-stats ./dada2/abx_denoising-stats.qza
```

Taking a long time.




#### Summarize the denoising   

No metadata file    

```
qiime feature-table summarize \
  --i-table ./dada2/abx_table.qza \
  --o-visualization ./dada2/abx_table_viz.qzv
qiime feature-table tabulate-seqs \
  --i-data ./dada2/abx_repseqs.qza \
  --o-visualization ./dada2/abx_repseqs_viz.qzv
```



#### Visualize the denoising stats   





```
qiime metadata tabulate \
  --m-input-file ./dada2/abx_denoising-stats.qza \
  --o-visualization ./dada2/abx_denoising-stats_viz.qzv
```



## Taxonomic Classification

#### Import classifiers for training datasets

Following

https://docs.qiime2.org/2023.5/data-resources/


> We also provide pre-formatted SILVA reference sequence and taxonomy files here that were processed using RESCRIPt. See licensing information below if you use these files.
Please cite the following references if you use any of these pre-formatted files:

> Michael S Robeson II, Devon R ORourke, Benjamin D Kaehler, Michal Ziemski, Matthew R Dillon, Jeffrey T Foster, Nicholas A Bokulich. RESCRIPt: Reproducible sequence taxonomy reference database management for the masses. bioRxiv 2020.10.05.326504; doi: https://doi.org/10.1101/2020.10.05.326504

> See the SILVA website for the latest citation information for SILVA.

> Note

> The Silva reference files provided here include species-level taxonomy. While Silva annotations do include species, Silva does not curate the species-level taxonomy so this information may be unreliable. In a future version of QIIME 2 we will no longer include species-level information in our Silva reference files. This is discussed on the QIIME 2 Forum here (see Species-labels: caveat emptor!).

> License Information:
The pre-formatted SILVA reference sequence and taxonomy files above are available under a Creative Commons Attribution 4.0 License (CC-BY 4.0). See the SILVA license for more information.

> The files above were downloaded and processed from the SILVA 138 release data using the RESCRIPt plugin and q2-feature-classifier. Sequences were downloaded, reverse-transcribed, and filtered to remove sequences based on length, presence of ambiguous nucleotides and/or homopolymer. Taxonomy was parsed to generate even 7-level rank taxonomic labels, including species labels. Sequences and taxonomies were dereplicated using RESCRIPt. Sequences and taxonomies representing the 515F/806R region of the 16S SSU rRNA gene were extracted with q2-feature-classifier, followed by dereplication with RESCRIPt.

```
mkdir /uufs/chpc.utah.edu/common/home/kapheim-group2/iguanas/green_iguana_antibiotics/training-feature-classifiers
cd training-feature-classifiers
wget https://www.arb-silva.de/fileadmin/silva_databases/qiime/Silva_132_release.zip
unzip Silva_132_release.zip
rm -R __MACOSX/
rm Silva_132_release.zip
# sequences
wget https://data.qiime2.org/2023.5/common/silva-138-99-seqs.qza
# taxonomy
wget https://data.qiime2.org/2023.5/common/silva-138-99-tax.qza
```

#### Training on Silva 138 database




*Extract reference reads*

>It has been shown that taxonomic classification accuracy of 16S rRNA gene sequences\
 improves when a Naive Bayes classifier is trained on only the region of the target \
 sequences that was sequenced (Werner et al., 2012). This may not necessarily \
 generalize to other marker genes (see note on fungal ITS classification below). \
 We know from the Moving Pictures tutorial that the sequence reads that we?re trying \
 to classify are 120-base single-end reads that were amplified with the 515F/806R \
 primer pair for 16S rRNA gene sequences. We optimize for that here by extracting \
 reads from the reference database based on matches to this primer pair, and then \
 slicing the result to 120 bases.

```
cd /uufs/chpc.utah.edu/common/home/kapheim-group2/iguanas/green_iguana_antibiotics
qiime feature-classifier extract-reads --i-sequences ./training-feature-classifiers/silva-138-99-seqs.qza \
  --p-f-primer GTGYCAGCMGCCGCGGTAA \
  --p-r-primer GGACTACNVGGGTWTCTAAT \
  --p-min-length 100 \
  --p-max-length 400 \
  --o-reads ./training-feature-classifiers/silva_138_99_refseqs.qza
```


This step took a long time (~20 min) to complete.

*Train the classifier*



```
qiime feature-classifier fit-classifier-naive-bayes \
  --i-reference-reads ./training-feature-classifiers/silva_138_99_refseqs.qza \
  --i-reference-taxonomy ./training-feature-classifiers/silva-138-99-tax.qza \
  --o-classifier ./training-feature-classifiers/silva_138_99_classifier.qza
```

This took a long time.
 
#### Classify rep sequences


> Verify that the classifier works by classifying the representative sequences \
> and visualizing the resulting taxonomic assignments.


The classify-sklearn step took a long time .

```
mkdir /uufs/chpc.utah.edu/common/home/kapheim-group2/iguanas/green_iguana_antibiotics/classified
qiime feature-classifier classify-sklearn \
  --i-classifier ./training-feature-classifiers/silva_138_99_classifier.qza \
  --i-reads ./dada2/abx_repseqs.qza \
  --o-classification ./classified/abx_SILVA_138_99_taxonomy.qza
qiime metadata tabulate \
  --m-input-file ./classified/abx_SILVA_138_99_taxonomy.qza \
  --m-input-file ./dada2/abx_repseqs.qza \
  --o-visualization ./classified/abx_SILVA_138_99_taxonomy_viz.qzv
```


## Generate a phylogenetic tree de novo


Use existing pipelines to generate a de novo tree that includes all sequences.

https://docs.qiime2.org/2023.5/tutorials/phylogeny/

[Scroll down to 'Pipelines' near the bottom]

> This pipeline will start by creating a sequence alignment using MAFFT, after \
which any alignment columns that are phylogenetically uninformative or ambiguously \
aligned will be removed (masked). The resulting masked alignment will be used to \
infer a phylogenetic tree and then subsequently rooted at its midpoint. Output files\
 from each step of the pipeline will be saved. This includes both the unmasked and \
masked MAFFT alignment from q2-alignment methods, and both the rooted and unrooted \
phylogenies from q2-phylogeny methods.

```
mkdir /uufs/chpc.utah.edu/common/home/kapheim-group2/iguanas/green_iguana_antibiotics/phylogeny
qiime phylogeny align-to-tree-mafft-fasttree \
    --i-sequences ./dada2/abx_repseqs.qza \
    --output-dir ./phylogeny/merged_mafft-fasttree-output_SILVA138
```

## Statistical analysis


Export to R and analyze in RStudio on laptop.

File `green_iguana_antibiotics.Rmd`

Exported:

| element | file |
| --- | --- |
| table | ./dada2/abx_table.qza |
| tree | ./phylogeny/merged_mafft-fasttree-output_SILVA138/rooted_tree.qza |
| taxonomy | ./classified/abx_SILVA_138_99_taxonomy.qza |
| metadata | ? |




 
