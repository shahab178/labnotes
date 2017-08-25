# Rumen microbiome technology bias check
---
*8/9/2017*

These are my notes and commands for testing the bias of different sequencing platforms in the collection and determining if there are major limitations to each.

## Mash profile comparisons

I am going to test for kmer cardinality differences in the methods using MASH. My concern is that certain kmers may be underrepresented in the different methods and this is the best method for finding out!

Some oddities of MASH: it doesn't read gzipped files and each separate file is considered an "island" in each sketch comparison. I need to merge files to keep them in the same profile for comparison. The great news is that the sketches I create can be transferred and used on other systems, making them great portable compressed profiles.

#### Fastq merger and preparation

```bash
cat *.fastq > nanopore_yu_and_morrison_3.combined.fq
for i in illuminaR*/*.fastq.gz; do echo $i; gunzip $i; done

# I wrote a python script to filter the reads
python3 ../../../programs_source/python_toolchain/sequenceData/filterNextseqFastqFiles.py -f YMPrepCannula_S1_L001_R1_001.fastq -r YMPrepCannula_S1_L001_R2_001.fastq -o YMRewriteFilterGbase
Processed 29316700 reads and identified 884938 single G-base and 3990662 both G-base artifacts

# Automating the rest
for i in illuminaR1/YMPrepCannula_S1_L002_R1_001.fastq,illuminaR1/YMPrepCannula_S1_L002_R2_001.fastq illuminaR1/YMPrepCannula_S1_L003_R1_001.fastq,illuminaR1/YMPrepCannula_S1_L003_R2_001.fastq illuminaR1/YMPrepCannula_S1_L004_R1_001.fastq,illuminaR1/YMPrepCannula_S1_L004_R2_001.fastq illuminaR2/YMPrepCannula_S1_L001_R1_001.fastq,illuminaR2/YMPrepCannula_S1_L001_R2_001.fastq illuminaR2/YMPrepCannula_S1_L002_R1_001.fastq,illuminaR2/YMPrepCannula_S1_L002_R2_001.fastq illuminaR2/YMPrepCannula_S1_L003_R1_001.fastq,illuminaR2/YMPrepCannula_S1_L003_R2_001.fastq illuminaR2/YMPrepCannula_S1_L004_R1_001.fastq,illuminaR2/YMPrepCannula_S1_L004_R2_001.fastq; do file1=`echo $i | cut -d',' -f1`; file2=`echo $i | cut -d',' -f2`; folder=`echo $i | cut -d'/' -f1`; echo $folder; lane=`basename $file1 | cut -d'_' -f3`; python3 ../../programs_source/python_toolchain/sequenceData/filterNextseqFastqFiles.py -f $file1 -r $file2 -o ${folder}"/YMRewriteFilterGbase."${lane} ; done
illuminaR1
Processed 52887277 reads and identified 1863215 single G-base and 26864810 both G-base artifacts
illuminaR1
Processed 26327191 reads and identified 572981 single G-base and 1135473 both G-base artifacts
illuminaR1
Processed 28095512 reads and identified 793048 single G-base and 3164439 both G-base artifacts
illuminaR2
Processed 63812154 reads and identified 808176 single G-base and 34846 both G-base artifacts
illuminaR2
Processed 63221220 reads and identified 892873 single G-base and 85939 both G-base artifacts
illuminaR2
Processed 63346742 reads and identified 777543 single G-base and 17110 both G-base artifacts
illuminaR2
Processed 62625319 reads and identified 818123 single G-base and 34208 both G-base artifacts
```

#### Mash sketch generation

I want to create Mash sketches that conform with Serge's metagenomics estimates and are comparable to the sketches I've generated [previously](https://github.com/njdbickhart/labnotes/blob/master/project_notes/metagenomics/Notes_20161219_dbick_metagenomics_software_test_notes.md#mash). So I will use the same settings as before.

```bash
for i in *.fastq; do name=`echo $i | cut -d'.' -f1`; echo $name; mash sketch -p 3 -k 21 -s 10000 -r -m 2 -o $name $i; done

for i in cheryl/*/*.subreads.bam; do name=`basename $i | cut -d'.' -f1`; echo $name; samtools fastq $i > $name.fq; mash sketch -p 3 -k 21 -s 10000 -r -m 2 -o $name $name.fq; rm $name.fq; done

# Sketching all Illumina reads from stdin
for i in illuminaR*/YMRewriteFilterGbase*fq; do cat $i; done | mash sketch -p 3 -k 21 -s 10000 -r -m 2 -o illumina_filtered_non-interleaved -

# Sketching pacbio reads separately from bam
for i in *.bam; do name=`echo $i | cut -d'.' -f1`; echo $name; samtools fastq $i | mash sketch -p 3 -k 21 -s 10000 -r -m 2 -o $name - ; done

for i in ./*.bam; do name=`basename $i | cut -d'.' -f1`; echo $name; samtools fastq $i | mash sketch -p 3 -k 21 -s 10000 -r -m 2 -o $name - ; done

# It's also only fair that I create a combined sketch from each facility's instrument for comparison
# Tim's first (RSII and Sequel)
cat LIB*.fastq | mash sketch -p 3 -k 21 -s 10000 -r -m 2 -o pacbio_rsii_summary -
for i in ./*/m54033*.bam; do samtools fastq $i; done | mash sketch -p 3 -k 21 -s 10000 -r -m 2 -o pacbio_sequel_tim_summary -

# Cheryl's second (Sequel and CCS'ed reads)
find ./ -name '*.bam' | grep -v 'scraps' | grep -v 'm54033' | xargs -I{} samtools fastq {} | mash sketch -p 3 -k 21 -s 10000 -r -m 2 -o pacbio_sequel_cheryl_summary -
cat ./*/*.fastq | mash sketch -p 3 -k 21 -s 10000 -r -m 2 -o ../pacbio_ccs_cheryl_summary -

# Nanopore read sketching is relatively straightforward; all of my generated data will be sketched together
for i in `find ./nanopore -name "*.fastq"`; do cat $i ; done | mash sketch -p 3 -k 21 -s 10000 -r -m 2 -o nanopore_yu_morrison_fastq -

```


## Mash sketch generation of outgroups

I want to include the cattle manure sample studies and the [Hess et al](https://www.ncbi.nlm.nih.gov/pubmed/21273488) dataset as "outgroups" to compare with our new dataset MASH profiles.

> Assembler2: /mnt/nfs/nfs2/bickhart-users/metagenomics_projects/

```bash
# The Hess et al. survey
sbatch --nodes=1 --ntasks-per-node=6 --mem=10000 --partition=assemble3 --wrap='for i in SRR094166_1.fastq.gz SRR094166_2.fastq.gz SRR094403_1.fastq.gz SRR094403_2.fastq.gz SRR094405_1.fastq.gz SRR094405_2.fastq.gz SRR094415_1.fastq.gz SRR094415_2.fastq.gz SRR094416_1.fastq.gz SRR094416_2.fastq.gz SRR094417_1.fastq.gz SRR094417_2.fastq.gz SRR094418_1.fastq.gz SRR094418_2.fastq.gz SRR094419_1.fastq.gz SRR094419_2.fastq.gz SRR094424_1.fastq.gz SRR094424_2.fastq.gz SRR094427_1.fastq.gz SRR094427_2.fastq.gz SRR094428_1.fastq.gz SRR094428_2.fastq.gz SRR094429_1.fastq.gz SRR094437_1.fastq.gz SRR094437_2.fastq.gz SRR094926_1.fastq.gz SRR094926_2.fastq.gz; do gunzip -c datasources/$i ; done | /mnt/nfs/nfs2/bickhart-users/binaries/mash-Linux64-v1.1.1/mash sketch -p 5 -k 21 -s 10000 -r -m 2 -o illumina_hess_combined - '

# Manure sample survey
sbatch --nodes=1 --ntasks-per-node=6 --mem=10000 --partition=assemble3 --wrap='for i in SRR2329878_1.fastq.gz SRR2329878_2.fastq.gz SRR2329910_1.fastq.gz SRR2329910_2.fastq.gz SRR2329939_1.fastq.gz SRR2329939_2.fastq.gz SRR2329962_1.fastq.gz SRR2329962_2.fastq.gz; do gunzip -c datasources/$i; done | /mnt/nfs/nfs2/bickhart-users/binaries/mash-Linux64-v1.1.1/mash sketch -p 5 -k 21 -s 10000 -r -m 2 -o illumina_manure_combined - '

# I transferred all of the files to my linux VM for distance comparison
```

## Mash distance estimation and concatenation

OK, now to generate the distance matrix and to create ordination plots.

> pwd: /home/dbickhart/share/metagenomics/pilot_project/mash_profiles

```bash
ls illumina_filtered_non-interleaved.msh illumina_hess_combined.msh illumina_manure_combined.msh FNFAE24738.msh nanopore_yu_morrison_fastq.msh pacbio_ccs_cheryl_summary.msh pacbio_rsii_summary.msh pacbio_sequel_cheryl_summary.msh pacbio_sequel_tim_summary.msh > combined_summary_sketches.list

mash paste -l combined_summary_sketches combined_summary_sketches.list

for i in `cat combined_summary_sketches.list`; do echo $i; mash dist -p 3 -t combined_summary_sketches.msh $i > $i.combined.dist; done
mkdir combined_dist
mv *.dist ./combined_dist/

# Because most of my sketches were done based on standard input, I have to be creative in how I paste the distance matricies together
# You can see the file IDs in the mash "info" field. 
perl -lane 'open(IN, "< combined_dist/$F[0].combined.dist"); $h = <IN>; $d = <IN>; chomp $d; @dsegs = split(/\t/, $d); $dsegs[0] = $F[0]; print join("\t", @dsegs);' < combined_summary_sketches.list > combined_dist/combined_distance.matrix

# Now to read it in and do some plotting in R
```

My goal here is a rapid NMDS with the following simplistic categories:

* Illumina
* Nanopore
* PacBio
* Outgroup (combined manure profile)

```R
library(MASS)
library(vegan)

# just in case I need them
library(dplyr)
library(tidyr)

mash_data <- read.delim("combined_dist/combined_distance.matrix", header=FALSE)
entry_names <- mash_data[,1]
mash_data.format <- select(mash_data, -V1)
rownames(mash_data.format) <- entry_names
colnames(mash_data.format) <- entry_names

ord <- metaMDS(as.dist(mash_data.format), trymax=100)
# Only took 20 iterations

# Base Vegan plotting polymorhpic functions did not work on the data. Had to hack out the NMDS coords
plot(scores(ord), col=c("red", "blue", "blue", "green", "red", "purple", "purple", "purple", "purple"), pch= c(15, 15, 16, 15, 16, 16, 17, 15, 15))
legend("bottomleft", c("UKNano", "USIllum", "Hess", "Manure", "USNano", "PBCCS", "PBRSII", "PBCheryl", "PBTim"), col=c("red", "blue", "blue", "green", "red", "purple", "purple", "purple", "purple"), pch= c(15, 15, 16, 15, 16, 16, 17, 15, 15))
dev.copy2pdf(file="combined_profile_nmds_vegan.pdf", useDingbats=FALSE)
```