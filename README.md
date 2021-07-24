# EquCab3 Project Assets

This repo contains exported metadata and runtime information related to the 2019 UMN-Interval Bio "EquCab3" project.  This project consisted of alignment of next-gen WGS data for 549 individuals, and joint variant-calling across the entire population with two separate variant callers.

The content of these files is exported from Interval Bio's proprietary batch orchestration software.  These files serve to document the methods and detail of the processing, but are not guaranteed to be _runnable_ as software.

These exported translations of each batch job are in the `phase1`, `phase2a`, and `phase2b` directories.  Each file contains a label, batch inputs and outputs, and each "stage" of the batch.  Each stage also has its inputs and outputs indicated, and the recorded command line.

- - - -

## The pipelines

The processing was split into two phases, **alignment** and **joint variant calling**.  The following software packages were used:

* [bcftools 1.9 / htslib 0.7.17 / samtools 1.9](https://www.htslib.org/)
* [fastqc 0.11.8](http://www.bioinformatics.babraham.ac.uk/projects/fastqc/)
* [GATK 4.1.0.0](https://gatk.broadinstitute.org/)
* [Picard 2.18.27](https://broadinstitute.github.io/picard/)
* [Platypus 0.8.1](https://www.well.ox.ac.uk/research/research-groups/lunter-group/lunter-group/platypus-a-haplotype-based-variant-caller-for-next-generation-sequence-data)
* [Trimmomatic 0.38](http://www.usadellab.org/cms/?page=trimmomatic)


### Alignment (phase 1)

The alignment pipeline generally followed the Broad best practices pipeline: trimming the reads, aligning with `bwa mem`, deduplication, per lane-based base quality-score recalibration (BQSR), and finally individual variant calling.  Three variant callers were used, Platypus, GATK, and bcftools.  Additional pipeline outputs were generated as QC: fastqc, Picard CollectMultipleMetrics, and separating unpaired and unmapped reads.

```
For each sample:

                                                              /-> [GATK]
[concat sets] -> [trim] -> [align / deduplication] -> [BSQR] ---> [Platypus]
   \                                                    \     \-> [bcftools]
    \                                                    \
     [fastqc]                                             \-> [QC metrics]
```

The sequencing read data for the 549 samples were from multiple heterogenous sources, collected over several years with multiple generations of Illumina machines, different sequencing centers, and even different library preparation.  As such, the FASTQ data had a wide range in quality across the 549 samples, including single vs. paired-end reads, varying read lengths, inconsistent read quality distribution between samples, and even differing PHRED-encoding for the quality scores.

Within each sample, base quality score recalibration was performed by annotating reads with read groups based on the instrument, run, flowcell, and lane of the read data.  Where possible this information was obtained through introspection of the sequence ids in the FASTQ data, otherwise it was entered manually based on other documentation.  Samples obtained from the SRA had this data stripped, and so for these samples they were each treated as a single recalibration domain.


### Joint Calling (phase 2a / 2b)

In order to manage the compute load and complexity, the joint variant calling pipeline was managed in two parts: the samples were first split by chromosome, then each chromosome was joint variant called across all samples.

In the splitting step samtools was used to split of the 549 samples into the 33 canonical chromosomes, with the remaining unplaced scaffolds grouped into a single file.  This resulted in 18666 BAM files (549 samples * (33+1) "chromosomes").

In the joint variant calling step all the bam files for a given chromosome were variant called togther, by GATK and bcftools in GVCF format (ie. all genomic positions).  Platypus was shown to have exponential runtime growth with respect to number of samples so it was not used.  The resulting output was 68 GVCF files (34 chromosomes * 2 variant callers).

