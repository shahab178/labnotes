# Unmapped RNA-seq rumen epimural community
---
*5/3/2018*

These are my notes on the separation of unmapped reads from Wenli's rumen epimural sequencing project.

## Table of Contents


## Generating unmapped read fastqs

Here is the location of the unmapped read bam files generated by TopHat:

> /mnt/nfs/nfs2/bickhart-users/cattle_asms/Rumen_epimural_RNAseq

Let's index them and check them for unmapped segments.

> Assembler2: /mnt/nfs/nfs2/bickhart-users/cattle_asms/Rumen_epimural_RNAseq

```bash
module load samtools; for i in *.bam; do echo $i; samtools index $i; done


```