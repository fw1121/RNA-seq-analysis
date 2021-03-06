### Why I am testing again?

I know there are papers/posts comparing different RNA-seq pipelines. For example:  
[A benchmark for RNA-seq quantification pipelines](http://genomebiology.biomedcentral.com/articles/10.1186/s13059-016-0940-1)  
[Differential analyses for RNA-seq: transcript-level estimates improve gene-level inferences](http://f1000research.com/articles/4-1521/v1#T2)  
[“ALIGNMENT FREE” TRANSCRIPTOME QUANTIFICATION](https://sjcockell.me/2015/05/18/alignment-free-transcriptome-quantification/)  
and [many others](https://github.com/crazyhottommy/RNA-seq-analysis#blog-posts-on-kallisto)

However, I just got several RNA-seq data to play with, and I think it is a good time-point for me to get my hands wet on those RNA-seq quantification tools (especially those alignment-free ones) and get a **personal** idea of how different tools perform. I am not 
doing bench-marking, as one should simulate the RNA-seq reads by e.g. [polyester](https://github.com/alyssafrazee/polyester) to have 
the ground truth.

Choosing alignment based tools (such as tophat, STAR, bowtie, HISAT) or alignment free ones depends on the purpose of your study. `Salmon` and `kallisto` requires the reads "pesudo-map" to the transcriptome, so one has to provide a fasta file containing all the transcripts you want to quantify. Therefore, if you want to find novel transcripts, you probably should go with the alignment based methods. It is also shown recently that [Widespread intron retention diversifies most cancer transcriptomes](https://genomemedicine.biomedcentral.com/articles/10.1186/s13073-015-0168-9). If you want to do similar things, you need to use mappers with the genome (not the transcriptiome) as a reference. 

`kallisto` can output a pseudo-bam which can be useful for some people. `Salmon` will have the same functionality in the next release according to Rob.

Now, let's begin my analysis.

I skipped quality trimming for this analysis. As it was shown that trimming may not necessary to be a good thing:  
[Trimming of sequence reads alters RNA-Seq gene expression estimates](https://bmcbioinformatics.biomedcentral.com/articles/10.1186/s12859-016-0956-2)  
For assembly [On the optimal trimming of high-throughput mRNA sequence data](http://journal.frontiersin.org/article/10.3389/fgene.2014.00013/full).  
However, as a rule of thumb, one needs to always check reads quality of any sequencing data sets. e.g. [RSeQC: An RNA-seq Quality Control Package](http://rseqc.sourceforge.net/)


### Testing [Salmon](https://github.com/COMBINE-lab/salmon) for RNA-seq quantification

Download the binary(v0.6.0) by:

```bash
wget https://github.com/COMBINE-lab/salmon/releases/download/v0.6.0/SalmonBeta-0.6.0_DebianSqueeze.tar.gz
tar xvzf SalmonBeta-0.6.0_DebianSqueeze.tar.gz
```

Read the documentation on how to use it [here](http://salmon.readthedocs.io/en/latest/salmon.html#using-salmon)

To use Salmon in quasi-mapping-based mode, then you first have to build an Salmon index for your transcriptome.
[Ensemble](http://useast.ensembl.org/info/data/ftp/index.html) release 75 is the latest version for GRCh37 (hg19).
starting from release 76 it is GRCh38 (hg38)

Download the hg19 version of cDNA and non-coding RNA fasta:

```bash
wget ftp://ftp.ensembl.org/pub/release-75/fasta/homo_sapiens/cdna/Homo_sapiens.GRCh37.75.cdna.all.fa.gz  
wget ftp://ftp.ensembl.org/pub/release-75/fasta/homo_sapiens/ncrna/Homo_sapiens.GRCh37.75.ncrna.fa.gz
## merge together 

gunzip -c Homo_sapiens.GRCh37.75.cdna.all.fa.gz Homo_sapiens.GRCh37.75.ncrna.fa.gz > Homo_sapiens.GRCh37.75.cdna.ncrna.fa
```
First, you run the Salmon indexer:

>Default index --- The quasi index has been made the default type. This means that it is no longer necessary to provide the 
--type option to the index command. The fmd index remains enabled, but may be removed in a future version.

```bash
salmon index -t Homo_sapiens.GRCh37.75.cdna.ncrna.fa -i Homo_sapiens.GRCh37.75_quasi_index 
```
It finished in a little over 10 mins.

quantify the transcript:

The RNA-seq library I am analyzing is single-end stranded, reads from reverse strand. so I specified `-l SR`. read [salmon doc](http://salmon.readthedocs.io/en/latest/library_type.html) for different library types. I especially like the figure representation below:  
![](https://cloud.githubusercontent.com/assets/4106146/16705915/d3bfe62c-455e-11e6-8c3c-68f94a34a1ba.png)

```bash
salmon quant -p 10 -i ~/annotations/Homo_sapiens.GRCh37.75_quasi_index -l SR -r <(zcat 3R_S18_L002_R1_001.fastq.gz) -o 3R_transcripts_quant

salmon quant -p 10 -i ~/annotations/Homo_sapiens.GRCh37.75_quasi_index -l SR -r <(zcat 50R_S19_L002_R1_001.fastq.gz) -o 50R_transcripts_quant

salmon quant -p 10 -i ~/annotations/Homo_sapiens.GRCh37.75_quasi_index -l SR -r <(zcat WT_S17_L002_R1_001.fastq) -o WT_transcripts_quant
```
The quantification finishes within minutes!

### quantification in gene level
Salmon can also give gene-level quantification as long as feed a gtf file. In addtion, I specify the fragment length to be 200 bp (default) and standard deviation of 20 (default 80). Details [here](https://github.com/crazyhottommy/RNA-seq-analysis/blob/master/salmon_kalliso_STAR_compare.md#what-is-effective-length-of-the-feature). For paired-end RNA-seq, the fragment length distribution can be infered from the fastq files, but for single-end data, it needs to be specified. 

```bash
wget ftp://ftp.ensembl.org/pub/release-75/gtf/homo_sapiens/Homo_sapiens.GRCh37.75.gtf.gz

salmon quant -p 10 -i ~/annotations/Homo_sapiens.GRCh37.75_quasi_index -l SR -r <(zcat 3R_S18_L002_R1_001.fastq.gz) -o 3R_transcripts_quant -g ~/annotations/Homo_sapiens.GRCh37.75.gtf --fldMean 200 --fldSD 20

salmon quant -p 10 -i ~/annotations/Homo_sapiens.GRCh37.75_quasi_index -l SR -r <(zcat 50R_S19_L002_R1_001.fastq.gz) -o 50R_transcripts_quant -g ~/annotations/Homo_sapiens.GRCh37.75.gtf --fldMean 200 --fldSD 20

salmon quant -p 10 -i ~/annotations/Homo_sapiens.GRCh37.75_quasi_index -l SR -r <(zcat WT_S17_L002_R1_001.fastq.gz) -o WT_transcripts_quant -g ~/annotations/Homo_sapiens.GRCh37.75.gtf --fldMean 200 --fldSD 20
```
It is recommended using [tximport](http://bioconductor.org/packages/release/bioc/html/tximport.html) to get the gene-level quantification. I asked the difference between `tximport` and `salmon quant -g`.
The developer of salmon @[Rob Patro](https://twitter.com/nomad421) answered:
>Main diffs I can think of (1) in R   
(2) integrated with DESeq2  
(3) Can derive multi-sample effective gene lengths

Further reading [Differential analyses for RNA-seq: transcript-level estimates improve gene-level inferences](http://f1000research.com/articles/4-1521/v1)

### testing [kallisto](https://pachterlab.github.io/kallisto/starting) for quantification.
Build index first:

```bash
kallisto index -i Homo_sapiens.GRCh37.75.cdna.ncrna.kalisto.idx Homo_sapiens.GRCh37.75.cdna.ncrna.fa
```
It took me around 15 mins to build the index.

quantification:

`-l, --fragment-length=DOUBLE  Estimated average fragment length`  
`-s, --sd=DOUBLE               Estimated standard deviation of fragment length`
                              `(default: value is estimated from the input data)`
>In the case of single-end reads, the -l option must be used to specify the average fragment length. Typical Illumina libraries produce fragment lengths ranging from 180–200 bp but it’s best to determine this from a library quantification with an instrument such as an Agilent Bioanalyzer.

see a question by [James](https://twitter.com/JamesPBLloyd) [on the google group](https://groups.google.com/forum/#!searchin/kallisto-sleuth-users/single$20end/kallisto-sleuth-users/VPJfzL502bw/e2JDq7ezBgAJ)

>Common values for single end reads are insert length 200 and sd 20. If you have any better information, like the person who prepped the library or better yet, data from bioanalyzer that will of course be better. 

`kallisto` can take `.gz` files.

```bash

kallisto quant -t 10 -i ~/annotations/Homo_sapiens.GRCh37.75.cdna.ncrna.kalisto.idx -o 3R_kaliso_output --single -l 200 -s 20 3R_S18_L002_R1_001.fastq.gz

kallisto quant -t 10 -i ~/annotations/Homo_sapiens.GRCh37.75.cdna.ncrna.kalisto.idx -o 50R_kaliso_output --single -l 200 -s 20 50R_S19_L002_R1_001.fastq.gz

kallisto quant -t 10 -i ~/annotations/Homo_sapiens.GRCh37.75.cdna.ncrna.kalisto.idx -o WT_kaliso_output --single -l 200 -s 20 WT_S17_L002_R1_001.fastq.gz
```
Finished in ~6 mins. again, blazing fast as `Salmon` does.

### counts versus TPM/RPKM/FPKM
I will need to quote from [this](https://statquest.org/2015/07/09/rpkm-fpkm-and-tpm-clearly-explained/) blog post on explaning technical differences among RPKM, FPKM and TPM.

>These three metrics attempt to normalize for sequencing depth and gene length. Here’s how you do it for RPKM:

1. Count up the total reads in a sample and divide that number by 1,000,000 – this is our “per million” scaling factor.  
2. Divide the read counts by the “per million” scaling factor. This normalizes for sequencing depth, giving you reads per million (RPM)  
3. Divide the RPM values by the length of the gene, in kilobases. This gives you RPKM.

>FPKM is very similar to RPKM. RPKM was made for single-end RNA-seq, where every read corresponded to a single fragment that was sequenced. FPKM was made for paired-end RNA-seq

>TPM is very similar to RPKM and FPKM. The only difference is the order of operations. Here’s how you calculate TPM:

1. Divide the read counts by the length of each gene in kilobases. This gives you reads per kilobase (RPK).
2. Count up all the RPK values in a sample and divide this number by 1,000,000. This is your “per million” scaling factor.
3. Divide the RPK values by the “per million” scaling factor. This gives you TPM.

>So you see, when calculating TPM, the only difference is that you normalize for gene length first, and then normalize for sequencing depth second. However, the effects of this difference are quite profound.

>When you use TPM, the sum of all TPMs in each sample are the same. This makes it easier to compare the proportion of reads that mapped to a gene in each sample. In contrast, with RPKM and FPKM, the sum of the normalized reads in each sample may be different, and this makes it harder to compare samples directly.

Read the following posts as well:  
[What the FPKM? A review of RNA-Seq expression units](https://haroldpimentel.wordpress.com/2014/05/08/what-the-fpkm-a-review-rna-seq-expression-units/)    
[In RNA-Seq, 2 != 2: Between-sample normalization](https://haroldpimentel.wordpress.com/2014/12/08/in-rna-seq-2-2-between-sample-normalization/)   
R code from the above post: 

```r
countToTpm <- function(counts, effLen)
{
    rate <- log(counts) - log(effLen)
    denom <- log(sum(exp(rate)))
    exp(rate - denom + log(1e6))
}
 
countToFpkm <- function(counts, effLen)
{
    N <- sum(counts)
    exp( log(counts) + log(1e9) - log(effLen) - log(N) )
}
 
fpkmToTpm <- function(fpkm)
{
    exp(log(fpkm) - log(sum(fpkm)) + log(1e6))
}
 
countToEffCounts <- function(counts, len, effLen)
{
    counts * (len / effLen)
}
 
################################################################################
# An example
################################################################################
cnts <- c(4250, 3300, 200, 1750, 50, 0)
lens <- c(900, 1020, 2000, 770, 3000, 1777)
countDf <- data.frame(count = cnts, length = lens)
 
# assume a mean(FLD) = 203.7
countDf$effLength <- countDf$length - 203.7 + 1
countDf$tpm <- with(countDf, countToTpm(count, effLength))
countDf$fpkm <- with(countDf, countToFpkm(count, effLength))
with(countDf, all.equal(tpm, fpkmToTpm(fpkm)))
countDf$effCounts <- with(countDf, countToEffCounts(count, length, effLength))
```

My colleage @samir processed the same data using STAR-HTseq piepline, and it gives the raw counts for each gene. He is very skepitcal 
about transcript level quantification and would focus on gene-level for now. What adds the complexity a bit is that he used `gencode v19` as annotation and the gene name has a dot + digits in the end of each gene. e.g. `ENSG00000000003.10`  vs `ENSG00000000003` in gtf files downloaded from ensemble. I will need to get rid of the digits in the end.

I will need to convert the raw counts from the `STAR-HTseq` pipeline to TPM for comparison as `Salmon` and `kallisto` output TPM and estimated counts. Read the post: [convert counts to TPM](https://www.biostars.org/p/171766/). [Kamil Slowikowski](https://gist.github.com/slowkow/c6ab0348747f86e2748b) wrote a function to convert counts to TPM, and the function involves an effective length of the features.

### what is effective length of the feature?

I traced back to this paper by Lior pachter group [Transcript assembly and quantification by RNA-Seq reveals unannotated transcripts and isoform switching during cell differentiation](http://www.nature.com/nbt/journal/v28/n5/full/nbt.1621.html).

It is quite mathematical, but the [general idea is](https://groups.google.com/forum/#!searchin/kallisto-sleuth-users/Effective$20Length/kallisto-sleuth-users/SlJWXFMEEiM/ftkrtPZyAQAJ):

>If we take the fragment length to be fixed, then the effective length is how many fragments can occur in the transcript. This turns out to be length - frag_len +1. The number of fragments coming from a transcript will be proportional to this number, regardless of whether you sequenced one or both ends of the fragment. In turn, when it comes to probabilistically assigning reads to transcripts the effective length plays a similar role again. Thus for short transcripts, there can be quite a difference between two fragment lengths.
To go back to your example if you have transcript of length 310, your effective length is 10 (if fragment length is 300) or 160 (if fragment length is 150) in either case, which explains the discrepancy you see.

From @Rob
>The effective length is computed by using the fragment length distribution to determine the effective number of positions that can be sampled on each transcript. You can think of this as convolving the fragment length distribution with the characteristic function (the function that simply takes a value of 1) over the transcript. For example if we observe fragments of length 50 --- 1000, a position more than 1000 bases from the end of the transcript will contribute a value of 1 to the effective length, while a position 150 bases will contribute a value of F(150), where F is the cumulative distribution function of the fragment length distribution. For **single end data, where we can't learn an empirical FLD**, we use a gaussian whose mean and standard deviation can be set with --fldMean and --fldSD respectively.

From [Harold Pimentel](https://twitter.com/hjpimentel)'s post above. He is in Lior Pachter's group.
> Effective length refers to the number of possible start sites a feature could have generated a fragment of that particular length. In practice, the effective length is usually computed as:
![](https://s0.wp.com/latex.php?latex=%5Cwidetilde%7Bl%7D_i+%3D+l_i+-+%5Cmu_%7BFLD%7D+%2B+1&bg=ffffff&fg=000000&s=0&zoom=2)

>where uFDL is the mean of the fragment length distribution which was learned from the aligned read. If the abundance estimation method you’re using incorporates sequence bias modeling (such as eXpress or Cufflinks), the bias is often incorporated into the effective length by making the feature shorter or longer depending on the effect of the bias.

### R scripts to convert HTseq counts to TPM

```{r}
library(dplyr)
setwd("/Volumes/mdarisngc03.mdanderson.edu/scratch/RNA-seq-26357338/")

list.files()

counts.3R<- read.table("STAR_3R-30390482_htseq.cnt", sep="\t", header=F, stringsAsFactors = F)
counts.50R<- read.table("STAR_50R-30393469_htseq.cnt", sep="\t", header=F, stringsAsFactors = F)
counts.WT<- read.table("STAR_WT-30393468_htseq.cnt", sep="\t", header=F, stringsAsFactors = F)

counts.df<- cbind(counts.WT, counts.3R, counts.50R)
counts.df<- counts.df[, c(1,2,4,6)]
names(counts.df)<- c("gene_name", "HTseq.WT", "HTseq.3R", "HTseq.50R")

## get rid of the digits in the end of the gene_name
counts.df<- counts.df %>% mutate(gene_name = gsub("\\.[0-9]+", "", gene_name)) 

```

To convert the raw counts from HTSeq (gencode v19 as annotation), I will need the length of each gene. Although one can compute the gene length from the gtf files, the gene-level output of `Salmon` has already computed it for me. I will just need to "borrow" from there.

```{r}
WT.Salmon.gene.quant<- read.table("./WT-30393468/WT_transcripts_quant/quant.genes.sf", sep="\t",
                                  header=T, stringsAsFactors = F)
gene.length<- dplyr::select(WT.Salmon.gene.quant, c(Name, Length))

counts.df<- inner_join(counts.df, gene.length, by =c("gene_name"="Name")) 
```

`counts_to_tpm` function from  https://www.biostars.org/p/171766/

```{r}
counts_to_tpm <- function(counts, featureLength, meanFragmentLength) {
  
  # Ensure valid arguments.
  stopifnot(length(featureLength) == nrow(counts))
  stopifnot(length(meanFragmentLength) == ncol(counts))
  
  # Compute effective lengths of features in each library.
  effLen <- do.call(cbind, lapply(1:ncol(counts), function(i) {
    featureLength - meanFragmentLength[i] + 1
  }))
  
  # Exclude genes with length less than the mean fragment length.
  idx <- apply(effLen, 1, function(x) min(x) > 1)
  counts <- counts[idx,]
  effLen <- effLen[idx,]
  featureLength <- featureLength[idx]
  
  # Process one column at a time.
  tpm <- do.call(cbind, lapply(1:ncol(counts), function(i) {
    rate = log(counts[,i]) - log(effLen[,i])
    denom = log(sum(exp(rate)))
    exp(rate - denom + log(1e6))
  }))

  # Copy the column names from the original matrix.
  colnames(tpm) <- colnames(counts)
  return(tpm)
}

featureLength<- counts.df$Length

## fragment length is 200bp 
meanFragmentLength<- c(200, 200, 200)

counts<- as.matrix(counts.df[,c(2,3,4)])
rownames(counts)<- counts.df$gene_name

TPM.from.HTSeq<- counts_to_tpm(counts, featureLength, meanFragmentLength)

```

### Import salmon and kallisto transcript level 

Use `tximport` to import `salmon` and `kallisto` transcript level quantification

```{r}
#source("https://bioconductor.org/biocLite.R")
#biocLite("EnsDb.Hsapiens.v75")

#create a tx2gene.txt table
library(EnsDb.Hsapiens.v75)
edb <- EnsDb.Hsapiens.v75

Tx.ensemble <- transcripts(edb,
          columns = c("tx_id", "gene_id", "gene_name"),
          return.type = "DataFrame")
nrow(Tx.ensemble)

tx2gene<- Tx.ensemble[,c(1,2)]

samples<- c("WT-30393468", "3R-30390482", "50R-30393469")
salmon.dir<- c("WT_transcripts_quant", "3R_transcripts_quant", "50R_transcripts_quant")
kallisto.dir<- c("WT_kaliso_output", "3R_kaliso_output", "50R_kaliso_output")

kallisto.files <- file.path(samples, kallisto.dir, "abundance.tsv")
names(kallisto.files)<- c("kallisto.WT", "kallisto.3R", "kallisto.50R")

salmon.files<- file.path(samples, salmon.dir, "quant.sf")
names(salmon.files)<- c("salmon.WT", "salmon.3R", "salmon.50R")

all(file.exists(kallisto.files))
all(file.exists(salmon.files))

library(tximport)
library(readr)

tx.kallisto <- tximport(kallisto.files, type = "kallisto", tx2gene = tx2gene, 
                        reader = read_tsv, countsFromAbundance = "lengthScaledTPM")
tx.salmon <- tximport(salmon.files, type = "salmon", tx2gene = tx2gene, 
                      reader = read_tsv, countsFromAbundance = "lengthScaledTPM")
```

### compare STAR-HTseq, kallisto and salmon
