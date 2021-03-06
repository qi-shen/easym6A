# easym6A: Process m6A/MeRIP-seq data in a single or batch job mode

[![release](https://img.shields.io/badge/release-v1.0.0-orange.svg)](https://img.shields.io/badge/release-v1.0.0-orange.svg)
[![license](https://img.shields.io/badge/license-GPLv3-green.svg)](https://img.shields.io/badge/license-GPLv3-green.svg)

easym6A creates a bash script to process m6A/MeRIP-seq data from adapter trimming to peak calling. It can also serve as a pipeline for RNA-seq data.

![easym6A_workflow.png](./easym6A_workflow.png)

## Dependencies:

- **Cutadapt**, tested with 1.15
- **Samtools**, tested with 1.7
- **HISAT2**, tested with 2.1.0
- **Picard**, tested with 2.17.10
- **bedtools**, tested with 2.27.1
- **bedGraphToBigWig**
- **StringTie**, tested with 1.3.4d
- **prepDE.py**
- **gffcompare**, tested with 0.10.4
- **MACS2**, tested with 2.1.1.20160309
- **HOMER**, tested with 4.9
- **R**, tested with 3.3.3. Install three libraries **exomePeak**, **MeTPeak** and **MeTDiff** in R.

## Installation:

Add paths of executable programs of above dependencies, easym6A.pl and 3peakSuit.R to PATH. Assume that their paths are listed as follows:

| Software  | Path  |
|------------------ |------------------------------------------ |
| Cutadapt  | ~/.local/bin  |
| Samtools  | ~/software/samtools-1.7/bin  |
| HISAT2  | ~/software/hisat2-2.1.0  |
| Picard  | ~/software/picard-2.17.10  |
| bedtools  | ~/software/bedtools-2.27.1/bin   |
| bedGraphToBigWig  | ~/software/ucsc  |
| StringTie   | ~/software/stringtie-1.3.4d.Linux_x86_64   |
| prepDE.py   | ~/software/stringtie-1.3.4d.Linux_x86_64   |
| gffcompare  | ~/software/gffcompare-0.10.4.Linux_x86_64  |
| MACS2   | ~/.local/bin  |
| HOMER   | ~/software/homer-4.9/bin   |
| R   | ~/software/R-3.3-el7-x86_64/bin  |
| easym6A.pl  | ~/software/easym6A   |
| 3peakSuit.R   | ~/software/easym6A   |

Here is an example of appending them in `~/.bash_profile` or `~/.profile`.

```bash
PATH=~/.local/bin:~/software/samtools-1.7/bin:~/software/hisat2-2.1.0:~/software/picard-2.17.10:~/software/bedtools-2.27.1/bin:~/home/shunliu/software/ucsc:~/software/stringtie-1.3.4d.Linux_x86_64:~/software/gffcompare-0.10.4.Linux_x86_64:~/software/homer-4.9/bin:~/software/R-3.3-el7-x86_64/bin:~/software/easym6A:$PATH

export PATH
```

Note that Line 1 and 8 of 3peakSuit.R are the paths of Rscript and installed packages, respetively. They are also required to be consistent with R related environment variables.

#### 3peakSuit.R Line 1

Modify `~/software/R-3.3-el7-x86_64/bin/Rscript` to meet with the path of Rscript.

```R
#!/usr/bin/env ~/software/R-3.3-el7-x86_64/bin/Rscript
```

#### 3peakSuit.R Line 8

Modify `~/software/R-3.3-el7-x86_64/lib64/R/library` to meet with the path of the R library where exomePeak, MeTPeak and MeTDiff are located.

```R
.libPaths("~/software/R-3.3-el7-x86_64/lib64/R/library")
```

## Usage:

### 1. Prepare the index for the reference genome and repetitive elements.

easym6A uses HISAT2 to map reads to the genome. One genome sequence file in fasta format and one genome annotation file in gtf format are required to build the index for the genome. More specification can be referred in the [HISAT2 manual](https://ccb.jhu.edu/software/hisat2/manual.shtml#the-hisat2-build-indexer).

```bash
# Given that hg38.fa and gencode.v27.annotation.gtf are in the directory geonome and annotation, respectively.

# Generate index files into genome/hg38_tran_index
mkdir -p genome/hg38_tran_index
# Extract exons from the gtf file
extract_exons.py annotation/gencode.v27.annotation.gtf > genome/hg38_tran_index/gencode.v27.annotation.exon
# Extract splicing sites from the gtf file
extract_splice_sites.py annotation/gencode.v27.annotation.gtf > genome/hg38_tran_index/gencode.v27.annotation.ss
# Build the index for the genome
hisat2-build --ss genome/hg38_tran_index/gencode.v27.annotation.ss --exon genome/hg38_tran_index/gencode.v27.annotation.exon genome/hg38.fa genome/hg38_tran_index/hg38_tran
```

As easym6A can filter reads mapped to repetitive elements, one repetitive element sequence file in fasta format is required to build the index for repetitive elements, unless the filter step is ignored.

```bash
# Given that one would like to filter reads mapped to rRNAs. hg38_rRNA.fa is in the directory genome.

# Generate index files into genome/hg38_rRNA_index
mkdir -p genome/hg38_rRNA_index
# Build the index for rRNAs
hisat2-build genome/hg38_rRNA.fa genome/hg38_rRNA_index/hg38_rRNA
```

### 2. Create a table file including m6A/MeRIP-seq sample information

The table file is consisted of 19 fields, which records basic information of samples such as their file names, file paths, library types, etc. One row for one sample. It can be edited in EXCEL and then saved as plain text format (.txt). Here is an example of four HeLa samples (two input-IP pairs, one for control, the another for treatment):

| Species | Cell Line | Dataset | Sample | Group | File Name | Fastq File Dir | Fastq File | Bam File Path | 5’ Adapter | 3’ Adapter | 5’ Barcode | 3’ Barcode | Q33 | Strandness | Fragment Length | Read Length | Seq Layout | ID |
|---------|-----------|---------|----------------|-----------------|----------------------|--------------------------|--------------------------------|----------------------------------------------|------------|------------|------------|------------|-----|------------|-----------------|-------------|------------|----|
| human | HeLa | - | siControl_rep1 | control_input | input_siControl_rep1 | /your/raw/data/file/path | input_siControl.fastq.gz | /your/bam/file/path/input_siControl_rep1.bam | - | - | - | - | N | R | 150 | 50 | SINGLE | 1 |
| human | HeLa | - | siControl_rep1 | control_ip | ip_siControl_rep1 | /your/raw/data/file/path | ip_siControl.fastq.gz | /your/bam/file/path/ip_siControl_rep1.bam | - | - | - | - | N | R | 150 | 50 | SINGLE | 2 |
| human | HeLa | - | siMETTL3_rep1 | treatment_input | input_siMETTL3_rep1 | /your/raw/data/file/path | input_siMETTL3.fastq.gz | /your/bam/file/path/input_siMETTL3_rep1.bam | - | - | - | - | N | R | 150 | 50 | SINGLE | 3 |
| human | HeLa | - | siMETTL3_rep1 | treatment_ip | ip_siMETTL3_rep1 | /your/raw/data/file/path | ip_siMETTL3.fastq.gz | /your/bam/file/path/ip_siMETTL3_rep1.bam | - | - | - | - | N | R | 150 | 50 | SINGLE | 4 |

- **Species:** Optional. Not used in the pipeline.
- **Cell Line:** Optional. Not used in the pipeline.
- **Dataset:** Optional. Not used in the pipeline.
- **Sample:** Optional. Not used in the pipeline.
- **Group:** Optional. Not used in the pipeline.
- **File Name:** Required. The name of the folder generated by easym6A.
- **Fastq File Dir:** Required. The directory path of the fastq file.
- **Fastq File:** Required. The name of the fastq file. For paired-end data, read 1 and read 2 file are seperated by comma.
- **Bam File Path:** Optional or required. It is optional if de novo data processing is chosen or required if one prefers to perform peak calling using exsiting bam files (related to the option `-k/--onlypeak`. See below).
- **5' Adapter:** Optional. The 5' adapter of the sequencing data. For paired-end data, 5' adapters of read 1 and read 2 are seperated by comma.
- **3' Adapter:** Optional. The 3' adapter of the sequencing data. For paired-end data, 3' adapters of read 1 and read 2 are seperated by comma.
- **5' Barcode:** Optional. The 5' barcode of the sequencing data. Note that the current version of easym6A does not support the option.
- **3' Barcode:** Optional. The 3' barcode of the sequencing data. Note that the current version of easym6A does not support the option.
- **Q33:** Required. Accepted value is either `Y` or `N`. `Y` for "Phred+33" quality encoding and `N` for "Phred+64" quality encoding.
- **Strandness:** Required. Accepted value is `F` (forward strand), `R` (reverse strand) or `U` (unstranded) for single-end data, `FR` (forward strand), `RF` (reverse strand) or `U` (unstranded) for paired-end data. These values are consistent with those in the HISAT2 option `--rna-strandness`.
- **Fragment Length:** Required. The insert size of the library. Used in peak calling tools.
- **Read Length:** Required. The sequencing read length of the data. Used in both steps of gene quantification and peak calling.
- **Seq Layout:** Required. Accepted value is either `SINGLE` for single-end data or `PAIRED` for paired-end data.
- **ID:** Required. Sample ID for numbering samples. An important field as easym6A uses it to recognize which samples to be run.

Note that leave `-` in optional fields if the information is not available. Other characters or strings are not accepted.

### 3. Create a configuration file including paths of required files and output.

The configuration file contains only two columns and 12 rows. The first column is the key word field that is not allowed to be modified. Edit the second column to tell easym6A the locations of input files that are expected to be loaded and running results that are expected to be output.

| Field | Path |
|----------------------|------------------------------------------------------|
| bash_log_dir | /log/output/directory |
| cutadapt_out_dir | /adapter/trimmed/data/output/directory |
| hisat2_index_repBase | /repetitive/elements/hisat2/index/directory/basename |
| hisat2_index_genome | /genome/sequences/hisat2/index/directory/basename |
| hisat2_out_dir | /alignment/results/output/directory |
| stringtie_out_dir | /gene/quantification/results/output/directory |
| chrom_size | /chromosome/sizes/filename |
| transcriptome_size | 100000000 |
| gtf_file | /gene/annotation/gtf/filename |
| bed12_file | /gene/annotation/bed12/filename |
| peak_out_dir | /peak/calling/results/output/directory |
| genome_fasta_file | /genome/sequences/fasta/filename |

- **bash_log_dir:** Set a directory for a bash script and a log file that records running status of the bash script.
- **cutadapt_out_dir:** Set a directory for adapter-trimmed data processed by Cutadapt.
- **hisat2_index_repBase:** The HISAT2 index path of repetitive elements. Consistent with the option `-x` in HISAT2.
- **hisat2_index_genome:** The HISAT2 index path of genome. Consistent with the option `-x` in HISAT2.
- **hisat2_out_dir:** Set a directory for alignment results. Library complexity statistics, normalized bedgraph and bigwig files for visualization are also in the directory.
- **stringtie_out_dir:** Set a directory for gene quantification resutls generated by StringTie. De novo assembly for novel transcripts, raw read counts of genes and transcripts and novel transcript statistics are also in the directory.
- **chrom_size:** The path of a two-column file for recording chromosome sizes. Used in bedGraphToBigWig and bedtools.
- **transcriptome_size:** It is calculated by summing up sizes of all exons in the annotation file (e.g. the gtf or bed12 file). Note that overlapped exons are mereged. Used in MACS2.
- **gtf_file:** The path of a gene annotation file in gtf format. Downloaded in GENCODE or Ensembl. Used in StringTie, gffcompare, exomePeak, MeTPeak and MeTDiff.
- **bed12_file:** The path of a gene annotation file in bed12 format. Converting the gtf file above to the file is recommanded. Used in bedtools.
- **peak_out_dir:** Set a directory for peak callling resutls. De novo motif enrichment analysis results are also in the directory.
- **genome_fasta_file:** The path of a genome sequence file in fasta format. Downloaded in UCSC or Ensembl. Used in bedtools.

### 4. Use easym6A

easym6A is a Perl script used to generate a specific bash script to process the m6A/MeRIP-seq data.

```
Options:
  -h/--help                   brief help message.
  --man                       full documentation.
  -s/--samplelist <file>      a table file that records sample infomation. (Required)
  -c/--configure <file>       a configuration file that records paths of required files and output. (Requried)
  -n/--sampleno <string>      a comma-seperated ID list of samples. (Required)
  -e/--runname <string>       a user-defined job name. (Required)
  -t/--threads <num>          number of CPU threads to run the pipeline. (Default: 1)
  -l/--parallel               reduce the analysis time in parallel mode. (Default: off)
  -m/--method <string>        name(s) of peak calling tool(s) that are used. (Default: all)
  -b/--batch <file>           a table file that records batch job information.
  -x/--bstart <int>           batch job start ID. Required when -b/--batch is set. Used together with -y/--bend.
  -y/--bend <int>             batch job end ID. Required when -b/--batch is set. Used together with -x/--bstart.
  -a/--onlybam                run gene expression analysis only. (Default: off)
  -u/--daq                    calculate gene expression in both of samples. (Default: off)
  -k/--onlypeak               run peak calling analysis only. (Default: off)
  -p/--rmrep                  remove reads from repetitive elements. (Default: off)
  -d/--rmdup                  remove reads from PCR duplication. (Default: off)
  -f/--keeptmp                keep intermediated files. (Default: off)
  -r/--run                    run bash script(s) generated by the pipeline. (Default: off)
```

- **-s/--samplelist:** Required. The path of a table file that includes m6A/MeRIP-seq sample information. Created in Step 2.
- **-c/--configure:** Required when the single job mode is used. The path of a configuration file that includes paths of required files and output. Created in Step 3.
- **-n/--sampleno:** Required when the single job mode is used. A comma-seperated ID list of samples. Sample IDs are indicated in the table file of Step 2. Note that the current version of easym6A can not analyze samples with replicates. Therefore, the option is always `ID,ID` (input,IP) for performing peak detection or `ID,ID,ID,ID` (contorl input,control IP,treatment input,treatment IP) for finding differential peaks. For instance, `1,2` represents that easym6A analyzes samples whose ID are 1 (input sample) and 2 (IP sample); `1,2,3,4` represents that easym6A analyzes samples whose ID are 1 (control input sample), 2 (control IP sample), 3 (treatment input sample) and 4 (treatment IP sample).
- **-e/--runname:** Required when the single job mode is used. Used for naming the bash script and log file..
- **-t/--threads:** Optional. The number of CPU threads is used to run the pipeline. Default: `1`.
- **-l/--parallel:** Optional. The current version of easym6A does not support the option.
- **-m/--method:** Optional. Accepted values are `exomePeak`, `MeTPeak`, 'MeTDiff' `MACS2` or `all`. A comma-seperated list is available to tell easym6A which tools are used together.  Note that `exomePeak`, `MeTPeak` and `MACS2`are used in peak detection. `exomePeak` and `MetDiff` are used in finding differential peaks. Default: `all`.
- **-b/--batch:** Required when the batch job mode is used. The path of a table file that includes batch job information. See below.
- **-x/--bstart:** Required when the batch job mode is used. The starting ID of a batch job. Used together with the option `-y/--bend`.
- **-y/--bend:** Required when the batch job mode is used. The ending ID of a batch job. Used together with the option `-x/--bstart`.
- **-a/--onlybam:** Optional. If it is specified, easym6A only generates alignment and gene quantification results, exactly as what RNA-seq data was processed. When easym6A was used to process RNA-seq data, `-a/--onlybam` should be specified together with `-u/--daq`. Default: off.
- **-u/--daq:** Optional. Originally, easym6A is designed to calculate gene quantification in the input sample but not in the IP sample of m6A/MeRIP-seq data. When `-u/--daq` is specified, gene expression is calculated in both of samples. It is useful when easym6A is regarded as a RNA-seq data processing pipeline. Default: off.
- **-k/--onlypeak:** Optional. If it is specified, easym6A only performs the step of peak calling by using exisiting bam files whose paths are in the sample information file. Default: off.
- **-p/--rmrep:** Optional. Remove reads mapped to repetitive elements first before they are mapped to genomes. If `-p/--rmrep` is specified, a HISAT2 index of repetitive elements is required. Default: off.
- **-d/--rmdup:** Optional. Remove reads from PCR duplication detected by Picard. Default: off.
- **-f/--keeptmp:** Optional. Keep intermediated files generated during data processing. Default: off.
- **-r/--run:** Run bash scripts generated by easym6A. Default: off.

After both of the table file and configuration file are created, easym6A can be used in one of the following modes:

#### Single job mode (call peaks):

```bash
# An example for de novo m6A/MeRIP-seq data processing.
easym6A.pl -s sampleList.txt -c configuration.txt -n 1,2 -e siControl -t 8 -r
# An example for de novo RNA-seq data processing.
easym6A.pl -s sampleList.txt -c configuration.txt -n 1,2 -e siControl -t 8 -a -u -r
# An example for performing peak calling only.
easym6A.pl -s sampleList.txt -c configuration.txt -n 1,2 -e siControl -t 8 -k -r
```

#### Single job mode (call differential peaks):

```bash
# An example for de novo m6A/MeRIP-seq data processing.
easym6A.pl -s sampleList.tx -c configuration.txt -n 1,2,3,4 -e siMETTL3 -t 8 -r
# An example for performing peak calling only.
easym6A.pl -s sampleList.txt -c configuration.txt -n 1,2,3,4 -e siMETTL3 -t 8 -k -r
```

#### Batch job mode:

```bash
# An example for de novo m6A/MeRIP-seq data processing.
easym6A.pl -s sampleList.txt -b batchList.txt -x 1 -y 10 -t 8 -r
```

A table file including batch job information is consisted of five fields. Here is an example of eight samples (input samples: 1,2,3 and 7; IP samples: 3,4,6 and 8).

| Sample ID | Config File Path | Job Name | Method | Batch ID |
|-----------|------------------|-----------|--------|----------|
| 1,3 | /config/file/path | ctrl_rep1 | all | 1 |
| 2,4 | /config/file/path | ctrl_rep2 | all | 2 |
| 5,6 | /config/file/path | ctrl_rep3 | all | 3 |
| 7,8 | /config/file/path | ctrl_rep4 | all | 4 |

- **Sample ID:** The same as the option `-n/--sampleno`.
- **Config File Path:** The same as the option `-c/--configure`.
- **Job Name:** The same as the option `-e/--runname`.
- **Method:** The same as the option `-m/--method`.
- **Batch ID:** Batch ID for numbering jobs. An important field as easym6A uses it to recognize which jobs to be run.

#### Options for peak calling

Options for peak calling are specified in Line 19 to 28 of `3peakSuit.R`. Modify these options before using `easym6A.pl`.

```R
peak_cutoff_pvalue <- 1e-3
peak_cutoff_fdr <- 0.05
window_width <- 50
sliding_step <- 10
minimal_mapq <- 20
fold_enrichment <- 2
diff_peak_cutoff_fdr <- 0.1
diff_peak_abs_fold_change <- 1.5
diff_peak_consistent_abs_fold_change <- 1.5
dup <- F
```

#### Difference between `easym6A_slurm.pl` and `easym6A_local.pl`

`easym6A_slurm.pl` is used for computer clusters where [Slurm](https://slurm.schedmd.com) is installed. Options for Slurm can be specified in Line 229 to 234 of the script. Change option values in these lines, delet some of these lines, or add new lines about Slurm parameter declaration are available in order to meet with the running requirement of your programes.

```bash
#SBATCH --job-name=m6Aseq
#SBATCH --output=$bash_log_dir/$run_name.log
#SBATCH --time=36:00:00
#SBATCH --partition=broadwl
#SBATCH --ntasks-per-node=24
#SBATCH --mem-per-cpu=2000
```

`easym6A_local.pl` is used for local servers without Slurm installation. It calls `nohup` to run the data analysis in the background.

### 4. Citation

If you use **easym6A** in published research, please cite:

Liu, S., Zhu, A., He, C., Chen M.. REPIC: a database for exploring the *N*<sup>6</sup>-methyladenosine methylome. Genome Biol **21**, 100 (2020). [https://doi.org/10.1186/s13059-020-02012-4](https://doi.org/10.1186/s13059-020-02012-4)
