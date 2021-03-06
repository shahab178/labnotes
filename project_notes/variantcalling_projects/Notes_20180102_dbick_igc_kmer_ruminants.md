# IGC kmer analysis in ruminants
---
*1/2/2018*

These are my notes on a kmer-based analysis of missing regions in ruminant genomes -- most immediately in goats.

## Table of Contents



## Goat Analysis

I selected a group of 100 goat WGS datasets from SRA using the default "goat" and "DNA" filters. The SRA entries were as follows:


In order to limit the amount of space the analysis files will take on the server, I am going to pipe the SRA content into jellyfish/Mash.

> Assembler2: /mnt/nfs/nfs2/bickhart-users/goat_projects

```bash
# Generating the mash profiles
perl -ne '$_ =~ s/\"//g; @s = split(/,/, $_); print join("\t", @s);' < goat_wgs_sra.csv > goat_wgs_sra.tab

perl -lane '@s = split(/,/, $F[0]); if($s[0] =~ /^Run/){next;} print $s[0]' < goat_wgs_sra_runifo.csv | xargs -I {} sbatch --nodes=1 --ntasks-per-node=2 --mem=5000 --wrap="fastq-dump -Z {} | /mnt/nfs/nfs2/bickhart-users/binaries/mash-Linux64-v1.1.1/mash sketch -b 1G -s 25000 -o {} - "

# Generating the jf databases
perl -lane '@s = split(/,/, $F[0]); if($s[0] =~ /^Run/){next;} print $s[0]' < goat_wgs_sra_runifo.csv | xargs -I {} sbatch --nodes=1 --ntasks-per-node=2 --mem=110000 -p assemble3 --wrap="module load sratoolkit/2.8.1-3; module load jellyfish/2.2.3; jellyfish count -s 3G -m 21 --bf-size 100G -t 10 -o {}.jf <(fastq-dump -Z {} )";

perl -lane '@s = split(/,/, $F[0]); if($s[0] =~ /^Run/){next;} print $s[0]' < goat_wgs_sra_runifo.csv | xargs -I {} sbatch --nodes=1 --ntasks-per-node=2 --mem=160000 -p assemble3 --wrap="module load sratoolkit/2.8.1-3; module load jellyfish/2.2.3; echo 'fastq-dump -Z {}' > {}.g; jellyfish count -s 3G -m 21 --bf-size 100G -t 10 -o {}.jf -g {}.g";

# Mashing some cattle profiles as an outgroup
sbatch dominette_hiseq_mash.sh
```

And here is the code in that simple script:

```bash
#!/usr/bin/bash
#SBATCH --nodes=1
#SBATCH --mem=15000
#SBATCH --ntasks-per-node=10
#SBATCH -p assemble3

for i in /mnt/nfs/nfs2/brosen/projects/ARS-UCD/Dominette_HiSeq_data/*.fastq.gz
do
        gunzip -c $i
done | /mnt/nfs/nfs2/bickhart-users/binaries/mash-Linux64-v1.1.1/mash sketch -b 1G -s 25000 -p 9 -o dominette_hiseq_mash -
```

Now to generate the distance matrix for mash and plot it as a PCA plot.

```bash
ls *.msh > combined_mash_sketches.list

/mnt/nfs/nfs2/bickhart-users/binaries/mash-Linux64-v1.1.1/mash paste -l combined_mash_sketches combined_mash_sketches.list
for i in `cat combined_mash_sketches.list`; do echo $i; /mnt/nfs/nfs2/bickhart-users/binaries/mash-Linux64-v1.1.1/mash dist -p 10 -t combined_mash_sketches.msh $i > ${i}.dist; done

# Now to generate a group assignment file
perl -ne '@b = split(/,/); print "$b[0]\t$b[10]\t$b[14]\t$b[19]\t$b[20]\t$b[24]\t$b[26]\t$b[29]\t$b[31]\t$b[33]\t$b[34]\t$b[40]\t$b[41]\n";' < goat_wgs_sra_runifo.csv > goat_wgs_sra_runifo.abbrev.tab

perl -e 'chomp(@ARGV); open(IN, "< $ARGV[0]"); @names; while(<IN>){chomp; @d = split(/\./); push(@names, $d[0]);} close IN; open(IN, "< $ARGV[1]"); <IN>; %h; while(<IN>){chomp; @s = split(/\t/); $h{$s[0]} = $s[7];} close IN; foreach my $b (@names){$d = ($h{$b} =~ /SAN/)? "SANEN" : "CASHMERE"; print "$b\t$h{$b}\t$d\n";}' combined_mash_sketches.list goat_wgs_sra_runifo.abbrev.tab > combined_mash_groups.tab

# I had to edit a few names to make it work. Now to make the distance matrix
perl -lane 'open(IN, "$F[0].dist"); $h = <IN>; $d = <IN>; chomp $d; @dsegs = split(/\t/, $d); @nm = split(/\./, $F[0]); $dsegs[0] = $nm[0]; print join("\t", @dsegs);' < combined_mash_sketches.list > combined_mash_distances.matrix
```

Loading things into R so that I can generate the plots and perform the analysis.

```R
library(vegan)
library(dplyr)
mash_data <- read.delim("combined_mash_distances.matrix", header=FALSE)
entries <- mash_data[,1]
mash_data.format <- select(mash_data, -V1)

rownames(mash_data.format) <- entries
colnames(mash_data.format) <- entries

groups <- read.delim("combined_mash_groups.tab", header=FALSE)
colnames(groups) <- c("run", "sample", "group")
groups$pch = c(16)
groups$pch[groups$group == "COW"] <- 17

groups$col[groups$group == "COW"] <- "black"
groups$col[groups$group == "CASHMERE"] <- "dodgerblue"
groups$col[groups$group == "SANEN"] <- "darkorange2"
groups$col[groups$group == "BOER"] <- "chartreuse3"

pcoa.result <- wcmdscale(mash_data.format, eig=TRUE, add="lingoes", k=75)

pdf(file="goat_pcoa_plot.pdf", useDingbats=FALSE)
plot(pcoa.result$points, xlab="PC1 (30.4%)", ylab="PC2 (2.5%)", type = "p", col=groups$col, pch = groups$pch, cex=1.4, asp = 1/1)
ordihull(pcoa.result$points, groups = groups$group, label = FALSE, col = c("black", "darkorange2", "dodgerblue", "chartreuse3"))
dev.off()
```

The results of the ordination showed a clear division between the different breeds, and some variability within them. I need to spruce up the quality of the plot, but it's a good start. Now to 

### KAT analysis of unique kmers

Rather than roll my own, I'm actually happy to see that someone has generated a tool for comparison of kmers in assemblies.

> Assembler2: /mnt/nfs/nfs2/bickhart-users/goat_projects

```bash
# Just a trial run here...
/mnt/nfs/nfs2/bickhart-users/binaries/KAT/kat-2.3.4/src/kat comp -t 15 -m 21 -n -p 'png' SRR3144624.jf SRR5557710.jf

# I want to see if any of the sequence datasets have unique kmers not present in the assembly
# first, unzip the assembly to a file
gunzip -c /mnt/nfs/nfs2/GoatData/Goat-Genome-Assembly/Papadum-v13/papadum-v13.full.fa.gz > ./ARS1.goat.reference.fasta
# generate the jellyfish database for goat
jellyfish count -s 3G -m 21 -t 20 -o ARS1.goat.reference.jf ./ARS1.goat.reference.fasta

# generating kmer count plots for each dataset
for i in *.jf; do name=`echo $i | cut -d'.' -f1`; echo $name; sbatch --nodes=1 --mem=45000 --ntasks-per-node=15 --wrap="/mnt/nfs/nfs2/bickhart-users/binaries/KAT/kat-2.3.4/src/kat comp -t 15 -o $name.kat $i ARS1.goat.reference.jf"; done

# Test spectra-mx plot
/mnt/nfs/nfs2/bickhart-users/binaries/KAT/kat-2.3.4/src/kat plot spectra-mx -i -p 'png' -o SRR5557728.kat-main.mx.spectra-mx.png SRR5557728.kat-main.mx

# A little more useful, but not by much
```

Unfortunately, the KAT analysis doesn't show too much useful information. I'll have to return to the drawing board here.


### Jellyfish unique kmer sets

My final idea is to grep out the unique kmers that map to the immune gene regions, map them back to the assembly and then estimate coverage differences to infer structural disparities. Let's try this from a reverse approach perspective: I will mask out the assembled IGC regions in ARS1 and then hash the reference and use that as a filter for the other jellyfish data.

> Assembler2: /mnt/nfs/nfs2/bickhart-users/goat_projects

```bash
# Preparing the reference for masking
module load samtools; samtools faidx ARS1.goat.reference.fasta

# Masking just the IG and LILR regions
maskFastaFromBed -fi ARS1.goat.reference.fasta -bed igc_regions_to_gap.bed -fo ARS1.goat.reference.igcmasked.fasta
module load jellyfish/2.2.3; jellyfish count -s 3G -m 21 -t 20 -o ARS1.goat.reference.igcmasked.jf ARS1.goat.reference.igcmasked.fasta

# Creating the masked goat background
mkdir tempsort
sbatch -p assemble1 --nodes=1 --mem=125000 --ntasks-per-node=5 --wrap="jellyfish dump ARS1.goat.reference.igcmasked.jf -c | awk '{print $1}' | sort -T /mnt/nfs/nfs2/bickhart-users/goat_projects/tempsort --parallel=4 -S 50% > ARS1.goat.reference.igcmasked.background"

# Damn, the awk command didn't work apparently. I need to remove the numbers so that diff can work downstream
# Also, I counted over 4 trillion possible kmers to be produced from a 21 mer! The file is going to be huge!

# Trying the filtration with KAT
```

OK, so KAT can filter the reads out of fastqs and leave only the fastq entries. I'll try to filter out the assembly "background" and realign the reads to the areas that I filtered away.

**TODO**: generate ref fasta from igc_regions_to_gap.bed; finish kat filtration script; run filter script; align post-filter reads to the igc_regions_to_gap fasta; count and normalize occurrences.

```bash
# Making a reference for alignment
perl -lane 'open(IN, "samtools faidx ARS1.goat.reference.fasta $F[0]:$F[1]-$F[2] |"); while(<IN>){chomp; if($_ =~ />/){print ">$F[3]";}else{print $_;}} close IN;' < igc_regions_to_gap.bed > ARS1.goat.igcregions.fasta

samtools faidx ARS1.goat.igcregions.fasta
bwa index ARS1.goat.igcregions.fasta

perl -lane '@s = split(/,/, $F[0]); if($s[0] =~ /^Run/){next;} print $s[0]' < goat_wgs_sra_runifo.csv | xargs -I {} sbatch download_sra_process_kat_filter.sh {}

# Oops! The script incorrectly guessed the kat output! 
perl -lane '@s = split(/,/, $F[0]); if($s[0] =~ /^Run/ || $s[0] =~ /SRR3144624/){next;} print $s[0]' < goat_wgs_sra_runifo.csv | xargs -I {} sbatch --nodes=1 --ntasks-per-node=2 --mem=25000 --wrap="bwa mem ARS1.goat.igcregions.fasta {}.kat.filter.kmer.in.fastq | samtools sort -T {}.temp -o {}.sorted.bam -"
```

And here is the script that I used to queue the jobs:

#### download_sra_process_kat_filter.sh

```bash
#!/usr/bin/bash
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=6
#SBATCH --mem=45000

# A one-shot script designed to download an SRA archive, filter it against a background kmer set and leave only the reads
# $1 = SRA accession
kat_folder=/mnt/nfs/nfs2/bickhart-users/binaries/KAT/kat-2.3.4/src/
base_dir=/mnt/nfs/nfs2/bickhart-users/goat_projects
reference=${base_dir}/ARS1.goat.igcregions.fasta
refjfdb=${base_dir}/ARS1.goat.reference.igcmasked.jf
fastq=${1}.fastq
kat_fastq=${1}.kat.filter.kmer.fastq
bam=${1}.sorted.bam

START=$(date +%s.%N)
module load sratoolkit/2.8.1-3 bwa samtools

# Download fastq
fastq-dump --split-spot $1

# Filter with KAT
$kat_folder/kat filter seq -t 6 -i -o ${1}.kat.filter.kmer --seq=$fastq $refjfdb
if [ -s $kat_fastq ]
then
        rm $fastq
else
        echo "did not find file $kat_fastq!"
fi

# align with bwa
bwa mem $reference $kat_fastq | samtools sort -T ${1}.temp -o $bam -
END=$(date +%s.%N)
DIFF=$(echo "$END - $START" | bc)
echo "Executed in $DIFF time from $START and $END"
```

OK, I now need to prepare the following analysis:
* MASH profiles of the filtered kmers
* Read depth matrix of the samples for cn.mops

Let's generate brute-force windows (flat, integer non-overlapping 500 bp windows) and then use them as our skeleton for the overlap.

```bash
# Mash profile generation for the filtered fastqs
for i in *.in.fastq; do name=`echo $i | cut -d'.' -f1`; echo $name; sbatch --nodes=1 --mem=5000 --ntasks-per-node=2 --wrap="cat $i | /mnt/nfs/nfs2/bickhart-users/binaries/mash-Linux64-v1.1.1/mash sketch -b 1G -s 25000 -o ${name}.filt -"; done

## Generating the distance matrix
ls *.filt.msh > combined_filt_mash_sketches.list
/mnt/nfs/nfs2/bickhart-users/binaries/mash-Linux64-v1.1.1/mash paste -l combined_filt_mash_sketches combined_filt_mash_sketches.list

# Generating distance files
for i in `cat combined_filt_mash_sketches.list`; do echo $i; /mnt/nfs/nfs2/bickhart-users/binaries/mash-Linux64-v1.1.1/mash dist -p 10 -t combined_filt_mash_sketches.msh $i > ${i}.dist; done

# and the final matrix file
perl -lane 'open(IN, "$F[0].dist"); $h = <IN>; $d = <IN>; chomp $d; @dsegs = split(/\t/, $d); @nm = split(/\./, $F[0]); $dsegs[0] = $nm[0]; print join("\t", @dsegs);' < combined_filt_mash_sketches.list > combined_filt_mash_distances.matrix

# Window generation
perl -lane 'for($x = 1; $x + 500 <= $F[1]; $x += 500){$e = $x + 499; print "$F[0]\t$x\t$e";}' < ARS1.goat.igcregions.fasta.fai > ARS1.goat.igcregions.windows.bed

# I'll now use bedtools coveragebed to read the bam and generate the data matrix needed for cn.mops
ls *.sorted.bam | xargs -I {} sbatch --nodes=1 --mem=15000 --ntasks-per-node=1 -p assemble3 --wrap="bedtools coverage -a ARS1.goat.igcregions.windows.bed -b {} -hist -sorted > {}.coverage.bed"

# That generated a list of regions that were zero depth and duplicated features in "A". I want simple counts
ls *.sorted.bam | xargs -I {} sbatch --nodes=1 --mem=15000 --ntasks-per-node=1 -p assemble3 --wrap="bedtools coverage -a ARS1.goat.igcregions.windows.bed -b {} -counts -sorted > {}.counts.bed"
```

Now to generate the "competing" mash PCoA plot.

```R
library(vegan)
library(dplyr)

mash_data <- read.delim("combined_filt_mash_distances.matrix", header=FALSE)
entries <- mash_data[,1]
mash_data.format <- select(mash_data, -V1)

rownames(mash_data.format) <- entries
colnames(mash_data.format) <- entries

groups <- read.delim("combined_mash_groups.tab", header=FALSE)
colnames(groups) <- c("run", "sample", "group")
groups$pch = c(16)
groups$pch[groups$group == "COW"] <- 17

groups$col[groups$group == "COW"] <- "black"
groups$col[groups$group == "CASHMERE"] <- "dodgerblue"
groups$col[groups$group == "SANEN"] <- "darkorange2"
groups$col[groups$group == "BOER"] <- "chartreuse3"

pcoa.result <- wcmdscale(mash_data.format, eig=TRUE, add="lingoes", k=75)

pdf(file="goat_filt_pcoa_plot.pdf", useDingbats=FALSE)
plot(pcoa.result$points, xlab="PC1 (58.8%)", ylab="PC2 (5.6%)", type = "p", col=groups$col, pch = groups$pch, cex=1.4, asp = 1/1)
ordihull(pcoa.result$points, groups = groups$group, label = FALSE, col = c("black", "dodgerblue", "chartreuse3", "darkorange2"))
legend("bottomright", legend=c("Cow", "Cashmere", "Sanen", "Boer"), col=c("black", "dodgerblue", "darkorange2", "chartreuse3"), pch=c(17,16,16,16), cex=1.4)
dev.off()
```


Generating the Cn.mops files.

> pwd: ~/share/side_projects/ruminant_kmer

```R
library("cn.mops")
library("GenomicRanges")
bedfiles <- list.files(pattern="+.bed$")
fileIds <- sapply(strsplit(bedfiles, ".", fixed = TRUE), "[[", 1)

rdtables <- lapply(bedfiles, function(bed){var <- read.delim(file=bed, sep="\t", header=FALSE, col.names=c("chr", "start", "end", "value")); fname <- sapply(strsplit(bed, ".", fixed = TRUE), "[[", 1); return(c(var, fname))})

coords <- data.frame(chr=rdtables[[1]]$chr, start=rdtables[[1]]$start, end=rdtables[[1]]$end)
GRangeObj <- as(coords, "GRanges")

rdVals <- data.frame(sapply(rdtables, "[[", 4))
colnames(rdVals) <- fileIds
for(i in colnames(rdVals)){mcols(GRangeObj)[i] <- rdVals[,i]}

res <- cn.mops(GRangeObj)
res <- calcIntegerCopyNumbers(res)

# There were 41 CNVRs and 502 CNVs among the datasets
# Let's print them out to see
write.table(as.data.frame(iniCall(res)), file="goat_inicalls.tab", sep="\t", quote=FALSE)
write.table(as.data.frame(cnvs(res)), file="goat_cnmops_cnvs.tab", sep="\t", quote=FALSE)
write.table(as.data.frame(cnvr(res)), file="goat_cnmops_cnvrs.tab", sep="\t", quote=FALSE)

# I saved the workspace for viewing the results later.

```

#### Generating additional sample-level comparisons using aligned sequence reads

I'm not giving up on drawing selective comparisons just yet. I want to see how disparate the clustering can be within the two different groups using k-means clustering.

> pwd: ~/share/side_projects/ruminant_kmer

```bash
# First I need to prepare the cn.mops output for further R modification
perl -ne '$_ =~ s/\"//g; $_ =~ s/CN(\d+)/$1/g; print $_;' < goat_cnmops_cnvrs.tab > goat_cnmops_cnvrs.format.tab

# Now to load it into R for kmeans clustering
```

Jumping right into an R session.

```R
cnvrs <- read.delim("goat_cnmops_cnvrs.format.tab")
library(dplyr)

cns <- select(cnvrs, -seqnames, -start, -end, -width, -strand)
cns <- t(cns)

# Calculating sum of variance
wss <- (nrow(cns) -1 ) * sum(apply(cns, 2, var))
for(i in 2:15) wss[i] <- sum(kmeans(cns, centers=i)$withinss)
plot(1:15, wss, type="b", xlab="Number of Clusters", ylab="Within groups sum of squares")
dev.copy2pdf(file="CN_scree_plot.pdf", useDingbats=FALSE)

# The scree plot suggests that 6 clusters may be optimal

fit <- kmeans(cns, 6)
aggregate(cns, by=list(fit$cluster), FUN=mean)
cns <- data.frame(cns, fit$cluster)

matrix <- dist(cns, method = "euclidean")
# Testing goodness of fit first
fit2 <- kmeans(cns, 5)
library(fpc)
cluster.stats(matrix, fit$cluster, fit2$cluster)

# The Dunn index was maximized and the Adjusted rand index was higher with 6 clusters than with 5, so we're good.
library(dendextend)
library(pvclust)
library(colorspace)
gdata <- read.delim("combined_mash_groups.tab", header=FALSE)
gdata <- filter(gdata, V3 != "COW")

fit <- as.dendrogram(hclust(matrix, method="ward.D"))

breed <- gdata$V3
cols <- rainbow_hcl(length(levels(breed)))

col_group <- cols[breed]
labels_colors(fit) <- col_group
col_group <- col_group[order.dendrogram(fit)]
plot(fit)
rect.dendrogram(fit, k=6, border = "red")
legend("topright", legend=unique(breed), col=c("blueviolet", "darkgoldenrod1", "pink"), pch=16, cex=1.4)
# Warning! The color order is borked in this plot! Damn R sorting!

# OK, we have a dendrogram with the features spread out by distance. Let's now try to get a PCA of the data as well
library(vegan)
my.rda <- rda(cns)
biplot(my.rda, display = c("sites"), type=c("points"))
ordihull(my.rda, group = breed, col = c(2,3,1))
legend("topright", col=c(2,3,1), lty = 1, legend = unique(breed))

# Pretty much showed the same results as the PCoA plot. Not too much divergence from breed

# One final diagram: a heatmap
library(RColorBrewer)
library(gplots)
heatmap.2(as.matrix(cns)[,c(-42)], Rowv = fit, Colv = NA, scale="none", col=brewer.pal(11,"Spectral"), RowSideColors=col_group, density.info="none", trace="none")
```