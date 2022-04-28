# CAPP_SEQ_pipeline

Documentation of pipeline for processing of hybrid capture end reads of WGS data. Pipeline accepts raw FASTQ forward and reverse reads and performs duplex UMI consensus and/or single strand UMI consensus variant calling.

![image](https://user-images.githubusercontent.com/92883998/157908176-ae0d0d86-72b1-4748-ac74-dd3d2ea2d08f.png)


# Required Packages

fastqc https://www.bioinformatics.babraham.ac.uk/projects/fastqc/

picard https://github.com/broadinstitute/picard

bwa http://bio-bwa.sourceforge.net/bwa.shtml

trimmomatic http://www.usadellab.org/cms/?page=trimmomatic

samtools http://samtools.sourceforge.net/

GATK4: https://gatk.broadinstitute.org/hc/en-us

fgbio:http://fulcrumgenomics.github.io/fgbio/


# Usage

Inputs:

``` Sample_ID``` 

``` Panel_ID``` 

```call_type```  Options = ```Duplex``` or ```Single``` Default = ```Duplex```

```min_read``` Default = ```1```

```READ_STRUCTURE``` Default = ```9M+T 9M+T```




Required Files:

``` ${Sample_ID}_R1.fastq.gz```  raw R1 FASTQ saved under "data/FASTQ"
``` ${Sample_ID}_R2.fastq.gz```  raw R2 FASTQ saved under "data/FASTQ"

``` ${Panel_ID}.baits.hs38DH.interval_list```   BAIT interval file saved under "data/panel"

``` ${Panel_ID}.targets.hs38DH.interval_list```   Target interval file saved under "data/panel"

```REFGENOME``` path to ucsc Homo_sapiens_assembly38.fasta

```BWA_INDEX``` path to hg38 bwa index

Optional:  ```PoN.csv``` pipeline output for PoNs saved under "data/PoN" 

The pipeline perform duplex read consensus calling and single-strand consensus calling using the ```call_type``` flag.

The # of single strand consensus reads required for dupelx calling can also be specificed using the ```min_read``` flag. Default = 1 see http://fulcrumgenomics.github.io/fgbio/tools/latest/CallDuplexConsensusReads.html

Read structure must be specified for extracting and grouping UMIs with ```READ_STRUCTURE``` flag. For more on read structure, see https://github.com/fulcrumgenomics/fgbio/wiki/Read-Structures


CAPP SEQ Metrics are computed using Picard ```CollectHsMetrics``` output to directory in csv format.

Variants are called using GATK ```HaplotypeCaller``` and ```Mutect2```

Variants combined and post-processed filtered in Python script with an optional PoNs filter and output in csv format.


# Pipeline Description
Identify the UMI read structure
  READ_STRUCTURE='9M+T 9M+T' #https://github.com/fulcrumgenomics/fgbio/wiki/Read-Structures


FASTQC generated from FASTQ files for paired-end reads (R1 & R2) using ```fastqc``` for <Sample_ID>

``` ${Sample_ID}_R1.fastq.gz```  raw R1 FASTQ saved under "data/FASTQ"
``` ${Sample_ID}_R2.fastq.gz```  raw R2 FASTQ saved under "data/FASTQ"


Convert FASTQ to unaligned BAM

     java -jar $FGBIO FastqToBam --input $FIRST_FASTQ $SECOND_FASTQ \
     --output $FGBIO_FASTQTOBAM_OUT --read-structures $READ_STRUCTURE --sort true --read-group-id $RG_PU --sample $SAMPLE_ID --platform-unit $RG_PU --library   $SAMPLE_ID
     
EXTRACT UMIs from unaligned BAM using read structure
  java -jar $FGBIO ExtractUmisFromBam -i $FGBIO_FASTQTOBAM_OUT -o $FGBIO_FASTQTOBAM_EXTRACTED_OUT -r $READ_STRUCTURE -t ZA ZB -s RX
 
ALIGN TO hg38 AND MERGE using bwa mem
  bwa mem -t 4 -M -w 2 -p -R $READGROUP $BWA_INDEX $PICARD_SAMTOFASTQ_OUT > $BWA_TEMP_SAM

  java -jar $PICARD MergeBamAlignment -UNMAPPED $FGBIO_FASTQTOBAM_OUT -ALIGNED $BWA_TEMP_SAM -O $PICARD_MERGEBAMALIGNMENT_OUT -R $REFGENOME -SO coordinate --ALIGNER_PROPER_PAIR_FLAGS true -MAX_GAPS -1 -ORIENTATIONS FR -VALIDATION_STRINGENCY SILENT -CREATE_INDEX true

GROUPED PAIRED READS BY UMI (Consensus Calling)
  java -jar $FGBIO GroupReadsByUmi -i $PICARD_MERGEBAMALIGNMENT_OUT -o $FGBIO_GROUPED_OUT -s paired --edits 0 --min-map-q $MIN_MAP_QUAL -f $UMI_CALLS_FAMILY_HISTOGRAM

  java -jar $FGBIO GroupReadsByUmi -i $PICARD_MERGEBAMALIGNMENT_OUT -o $FGBIO_GROUPED_OUT -s adjacency --edits 0 --min-map-q $MIN_MAP_QUAL -f UMI_CALLS_FAMILY_HISTOGRAM

CALL CONSENSUS READS based on $DUBLE_CONSENSUS (required reads for d

  java -jar $FGBIO CallDuplexConsensusReads -i $FGBIO_GROUPED_OUT -o $FGBIO_CALLDUPLEXCONSENSUS_GROUPED_OUT --error-rate-pre-umi 45 --error-rate-post-umi 40 --min-input-base-quality $MIN_MAP_QUAL --min-reads $DOUBLE_CONSENSUS $SINGLE_CONSENSUS $SINGLE_CONSENSUS

  java -jar $FGBIO CallMolecularConsensusReads -i $FGBIO_GROUPED_OUT -o $FGBIO_CALLDUPLEXCONSENSUS_GROUPED_OUT --error-rate-pre-umi 30 --min-reads $SINGLE_CONSENSUS --error-rate-post-umi 30 --min-input-base-quality $MIN_MAP_QUAL
