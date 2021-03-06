# Ruminant variant genome-wide comparison project
---
*11/2/2015*

These are my notes on the pan ruminant comparative genomics project I'm hoping to present at PAG. 

Let's start by first generating the BAMs I need. Buffalo will take some time so let's get that started.

> Blade2: /mnt/iscsi/md3200i_4/dbickhart/Buffalo/new_umd3

```bash
# Generating the spreadsheet
ls ../../../schroeder/Project_WaterBuffalo/full/ | perl -e '%h; while(<>){chomp; @s = split(/_/); $h{$s[0]}->{$s[4]}->{$s[3]} = $_;} foreach $an (keys(%h)){foreach $num (keys(%{$h{$an}})){ $fq1 = $h{$an}->{$num}->{"R1"}; $fq2 = $h{$an}->{$num}->{"R2"}; print "$fq1\t$fq2\t$an\t$an\n";}}' > new_umd3_spreadsheet.tab
```

Damn, I can't install my version of Perl on the system! A prior local-lib installation is interfering with everything and I don't have time to correct it. I'm going to have to make due with the cow4 alignments I guess!

OH! Good news! Thanks to my checking my old notes, Steve had apparently transferred the old Buffalo data to my QNAP drive! Nice! Let's get the spreadsheets ready on Blade14 instead (where I know my pipeline can run).

> Blade14: /home/dbickhart/vnx_gliu_7/ruminant_project

```bash
# Getting the buffalo spreadsheet ready
ls /mnt/cifs/bickhart-qnap/buffalo_sequence/Sample_*/*.gz | perl -e 'use File::Basename; %h; while(<>){chomp; $b = basename($_); @s = split(/_/, $b); $h{$s[0]}->{$s[4]}->{$s[3]} = $_;} foreach $an (keys(%h)){foreach $num (keys(%{$h{$an}})){ $fq1 = $h{$an}->{$num}->{"R1"}; $fq2 = $h{$an}->{$num}->{"R2"}; print "$fq1\t$fq2\t$an\t$an\n";}}' > buffalo_fq_spreadsheet.tab

# Now the goat
ls /mnt/nfs/nfs2/GoatData/African-Goat-Resequencing/NextSeq500/150820_NS500432_0014_AH15AFBGXX/H15AFBGXX/African-Goat-ReSeq/G0*/*.gz | perl -e 'use File::Basename; %h; while(<>){chomp; $b = basename($_); @s = split(/_/, $b); $h{$s[0]}->{$s[4]}->{$s[3]} = $_;} foreach $an (keys(%h)){foreach $num (keys(%{$h{$an}})){ $fq1 = $h{$an}->{$num}->{"R1"}; $fq2 = $h{$an}->{$num}->{"R2"}; print "$fq1\t$fq2\t$an\t$an\n";}}' > goat_spreadsheet.tab

# Just some minor housecleaning -- I don't want to waste time with EFF and SNP calling!
cat buffalo_fq_spreadsheet.tab goat_spreadsheet.tab > combined_spreadsheet.tab
cp ~/.mergedpipeline.cnfg ./temp_pipeline.cnfg


perl ~/perl_toolchain/sequence_data_pipeline/runMergedBamPipeline.pl --fastqs combined_spreadsheet.tab --output goat_buff_bams --reference ../reference/umd3_kary_unmask_ngap.fa --config temp_pipeline.cnfg --threads 20
```

OK, I had to remove the error message profiling (open3 wasn't working with BWA for some reason!) but I'm now aligning samples to UMD3. As far as the 100 bulls project, I have samples that are aligned by BWA (version 6.X, not 7.X) to the right reference genome, but with the wrong algorithm (SAMPE, not MEM). Let's lay out the overall plan:

#### Project/Manuscript workflow
* Identify MEI, misassembly and large scale genome divergence in Goat, Cattle and Buffalo
* Annotate genomic regions and identify gene families under expansion/contraction
* Focus on the transcriptional regulation regions with MEI discovery
* Additionally SNPs and INDELs in the regulatory regions
* Identify cis-regulatory elements that are conserved/altered between cattle/goat/buffalo
* Also focus on gene deletions
* Identify individual diversity profiles and remove them as potential CNV "noise"
* TARGET: PNAS if I can get enough data

#### 100 bulls data
* For the abstract: Run JaRMS on samples and identify misassembled regions
* Later: Run RAPTR-SV and pull trans-chr reads to identify MEI
* Use this data as the "background"

#### Buffalo and Goat
* Align to UMD3 and remove misassembled regions
	* Misassembled regions are defined as:
	* Regions that are deletions, but are not gaps and not repeats
* Run RAPTR-SV and identify trans-chr MEIs
* Estimate count of unmapped reads per genome (supplementary data)

## JaRMS test on 100 bulls data

OK I know that the BAM files for the 100 bulls samples follow this format within the 100_base_run folder:

> (sample_name)/(sample_name).rg.rd.full.sorted.merged.bam

I'm currently using 75% of the memory of the blade, so let's try looping through these files one at a time to get the preliminary data first.

> Blade14: /mnt/iscsi/vnx_gliu_7/100_base_run

```bash
# OK! Running! 
~/jdk1.8.0_05/bin/java -jar ~/JaRMS/store/JaRMS.jar call -i BIBR02/BIBR02.rg.rd.full.sorted.merged.bam -f ../reference/umd3_kary_unmask_ngap.fa -o BIBR02.jarms.calls -t 2

# NOTE: my equation for estimating the X coverage of the genome always underestimates it by a factor of 1 x 10^10. 
# I need to check what's wrong, adjust it, and add an option to manually set window size

# OK, I uploaded a new version of JaRMs, but I might just keep the 500bp windows (or lower them to 400?)
~/jdk1.8.0_05/bin/java -jar ~/JaRMS/store/JaRMS.jar call -i BIBR02/BIBR02.rg.rd.full.sorted.merged.bam -f ../reference/umd3_kary_unmask_ngap.fa -o BIBR02.jarms.400 -t 2 -w 400


for i in *cnvs.bed; do cat $i | ~/bin/bed_length_sum.pl; done
400bp wins
        Interval Numbers:       21171
        Total Length:           286758029
        Length Average:         13544.8504558122
        Length Median:          1199
        Min Length:             16399
        Max Length:             3999
        Length Stdev:           34721.7105164687
500bp wins
        Interval Numbers:       16882
        Total Length:           241222118
        Length Average:         14288.7168581922
        Length Median:          6749
        Min Length:             54499
        Max Length:             3999
        Length Stdev:           33876.4349035586

# That was the total, let's try just the duplications (I suspect the 400bp wins picked up more gaps)
for i in BIBR02.*cnvs.bed; do echo $i; grep 'duplication' $i | perl ~/bin/bed_length_sum.pl; done
BIBR02.jarms.400cnvs.bed
        Interval Numbers:       8848
        Total Length:           120026752
        Length Average:         13565.4104882459
        Length Median:          125599
        Min Length:             16399
        Max Length:             1999
        Length Stdev:           39448.5304808339
BIBR02.jarms.callscnvs.bed
        Interval Numbers:       6494
        Total Length:           87859006
        Length Average:         13529.2587003388
        Length Median:          2249
        Min Length:             54499
        Max Length:             1999
        Length Stdev:           31939.4778282759

# Hmmm, so window length really changes the proportion of calls. Let's see how many novel calls there are.

intersectBed -a BIBR02.jarms.callscnvs.bed -b BIBR02.jarms.400cnvs.bed -v | bed_length_sum.pl
        Interval Numbers:       5142
        Total Length:           29343858
        Length Average:         5706.70128354726
        Length Median:          2999
        Min Length:             2999
        Max Length:             2499
        Length Stdev:           8718.15676037973

# Interesting. At least 50% of the calls are 3kb length, but there are still some differences
intersectBed -b BIBR02.jarms.callscnvs.bed -a BIBR02.jarms.400cnvs.bed -v | bed_length_sum.pl
        Interval Numbers:       8862
        Total Length:           63116338
        Length Average:         7122.13247573911
        Length Median:          9199
        Min Length:             27999
        Max Length:             9199
        Length Stdev:           13124.7575031819

# In this case, all of the calls are smaller

# I'm going to go with a flat 500bp window for everything. This way, I can compare the low coverage results against the higher coverage ones.
# Let's automate this for now and then take care of the issues later.
for i in `ls -d B?????`; do echo $i; ~/jdk1.8.0_05/bin/java -jar ~/JaRMS/store/JaRMS.jar call -i $i/${i}.rg.rd.full.sorted.merged.bam -f ../reference/umd3_kary_unmask_ngap.fa -o ${i}.auto.jarms -t 2 -w 500; done
```

The deadline is approaching, so I need to get some very preliminary results in order to pad my abstract. I think that I'm going to take the JaRMS calls that have finished for the Indicus animals and compare them to the RAPTR-SV results I previously generated (March 2 2015). My hope is to tie tandem dups with predicted duplication regions -- non-duplicated tandem repeat signatures indicate assembly shuffling.

Let's talk about the signatures I'm looking for:

* Genomic region rearrangement - tandem dups without RD signal, or trans-chr alignments
* MEI - transchr or maxdist repetitive reads
* Genomic deletion  deletion present in all individuals

OK, some summary statistics before I begin:

> Blade14: /mnt/iscsi/vnx_gliu_7/100_base_run

```bash
wc -l raptr-sv/finalfilter/BI*.tand.bed
   166 raptr-sv/finalfilter/BIBR02.finalfilter.raptr.tand.bed
   616 raptr-sv/finalfilter/BIBR03.finalfilter.raptr.tand.bed
   132 raptr-sv/finalfilter/BIBR04.finalfilter.raptr.tand.bed
   615 raptr-sv/finalfilter/BIBR05.finalfilter.raptr.tand.bed
  1447 raptr-sv/finalfilter/BIBR07.finalfilter.raptr.tand.bed
   268 raptr-sv/finalfilter/BIBR08.finalfilter.raptr.tand.bed
   130 raptr-sv/finalfilter/BIBR09.finalfilter.raptr.tand.bed
    75 raptr-sv/finalfilter/BIGI01.finalfilter.raptr.tand.bed
   348 raptr-sv/finalfilter/BIGI02.finalfilter.raptr.tand.bed
   415 raptr-sv/finalfilter/BIGI05.finalfilter.raptr.tand.bed
   109 raptr-sv/finalfilter/BIGI06.finalfilter.raptr.tand.bed
  1085 raptr-sv/finalfilter/BIGI07.finalfilter.raptr.tand.bed
   347 raptr-sv/finalfilter/BIGI08.finalfilter.raptr.tand.bed
  6723 raptr-sv/finalfilter/BINE01.finalfilter.raptr.tand.bed
   376 raptr-sv/finalfilter/BINE04.finalfilter.raptr.tand.bed
   187 raptr-sv/finalfilter/BINE07.finalfilter.raptr.tand.bed
   203 raptr-sv/finalfilter/BINE09.finalfilter.raptr.tand.bed
   352 raptr-sv/finalfilter/BINE10.finalfilter.raptr.tand.bed
    61 raptr-sv/finalfilter/BINE12.finalfilter.raptr.tand.bed
   187 raptr-sv/finalfilter/BINE13.finalfilter.raptr.tand.bed
   250 raptr-sv/finalfilter/BINE23.finalfilter.raptr.tand.bed
 14092 total

# So, BINE1 is pretty bad. Let's focus on Brahman for the abstract
# Let's go through my analysis pipeline
intersectBed -a raptr-sv/finalfilter/BIBR02.finalfilter.raptr.tand.bed -b BIBR02.auto.jarmscnvs.bed | wc -l
	23
grep 'duplication' BIBR02.auto.jarmscnvs.bed | wc -l
	5075
wc -l raptr-sv/finalfilter/BIBR02.finalfilter.raptr.tand.bed
	166 raptr-sv/finalfilter/BIBR02.finalfilter.raptr.tand.bed

# That leaves about 143 tandem events unaccounted for
# Let's confirm by checking their supporting read counts
# I noticed that "3" was the mode for this sample
intersectBed -a raptr-sv/finalfilter/BIBR02.finalfilter.raptr.tand.bed -b BIBR02.auto.jarmscnvs.bed -v | perl -lane 'if($F[4] > 3){print $_;}' | wc -l
	36 <- 36 candidate rearrangements

# Let's get the numbers for the rest of the samples
for i in BIBR02 BIBR03 BIBR04 BIBR05 BIBR07 BIBR08 BIBR09; do echo -en "$i\t"; intersectBed -a raptr-sv/finalfilter/${i}.finalfilter.raptr.tand.bed -b ${i}.auto.jarmscnvs.bed -v | perl -lane 'if($F[4] > 3){print $_;}' | wc -l; done
BIBR02  36
BIBR03  Error: The requested bed file (BIBR03.auto.jarmscnvs.bed) could not be opened. Exiting!
0
BIBR04  32
BIBR05  95
BIBR07  Error: The requested bed file (BIBR07.auto.jarmscnvs.bed) could not be opened. Exiting!
0
BIBR08  76
BIBR09  32

# Weird! The BIBR03 and BIBR07 files have idx files... oh well. I'll debug later.
# Let's print out the data to text files so I can intersect them
for i in BIBR02 BIBR03 BIBR04 BIBR05 BIBR07 BIBR08 BIBR09; do echo -en "$i\n"; intersectBed -a raptr-sv/finalfilter/${i}.finalfilter.raptr.tand.bed -b ${i}.auto.jarmscnvs.bed -v | perl -lane 'if($F[4] > 3){print $_;}' > temp.${i}.unsupported.tand; done

# Ok, I'm testing by grepping out chromosomes and comparing by eye. Here are some I've found
# chr2
temp.BIBR02.unsupported.tand:chr2       65185285        65185604        TANDEM  4.0
temp.BIBR04.unsupported.tand:chr2       65185274        65185584        TANDEM  4.0
temp.BIBR08.unsupported.tand:chr2       65185312        65185642        TANDEM  8.0

# chr4
temp.BIBR02.unsupported.tand:chr4       6117380 6117469 TANDEM  11.0
temp.BIBR04.unsupported.tand:chr4       6117387 6117469 TANDEM  5.0
temp.BIBR08.unsupported.tand:chr4       6117381 6117467 TANDEM  9.0

# chr10
temp.BIBR02.unsupported.tand:chr10      26147295        26148209        TANDEM  5.0
temp.BIBR04.unsupported.tand:chr10      26147248        26148178        TANDEM  6.0
temp.BIBR08.unsupported.tand:chr10      26147232        26148206        TANDEM  4.0
```

Hmm, I just realized that many of these were smaller regions that would not be detected by JaRMS at a 500bp window size anyways...

## JaRMS on Goat and Buffalo

This is my first-pass test of JaRMS on the Goat and Buffalo samples. Let's see how it performs.

> Blade14: /mnt/iscsi/vnx_gliu_7/ruminant_project/goat_buff_bams

```bash
for i in `ls */*.bam`; do name=`echo $i | cut -d'/' -f1`; echo $name; ~/jdk1.8.0_05/bin/java -jar ~/JaRMS/store/JaRMS.jar call -i $i -f ../../reference/umd3_kary_unmask_ngap.fa -o ${name}.jarms -t 5 -w 500; done

wc -l *.bed
    5941 AG280.jarmscnvs.bed
    5875 AG304.jarmscnvs.bed
    8814 AG306.jarmscnvs.bed
   25795 ITWB10.jarmscnvs.bed
   26677 ITWB11.jarmscnvs.bed
   26853 ITWB12.jarmscnvs.bed
   24230 ITWB13.jarmscnvs.bed
   21375 ITWB14.jarmscnvs.bed
   25063 ITWB15.jarmscnvs.bed
   21298 ITWB1.jarmscnvs.bed
   23390 ITWB2.jarmscnvs.bed
   23708 ITWB3.jarmscnvs.bed
   23884 ITWB4.jarmscnvs.bed
   22783 ITWB5.jarmscnvs.bed
   17310 ITWB6.jarmscnvs.bed
   23141 ITWB7.jarmscnvs.bed
   24781 PC1.jarmscnvs.bed
  350918 total

# Quite a few more calls in Buffalo! Might be a dataset bias... but we can't focus on that for now
```

## RAPTR-SV on new Goat and Buffalo bams

I need to generate the RAPTR-SV calls for Goat and Buffalo. Let's do this in an automated fashion for now.

> Blade14: /mnt/iscsi/vnx_gliu_7/ruminant_project/goat_buff_bams

```bash
for i in `ls */*.bam`; do name=`echo $i | cut -d'/' -f1`; echo $name; ~/jdk1.8.0_05/bin/java -Xmx50g -jar ~/RAPTR-SV/store/RAPTR-SV.jar preprocess -i $i -o ${name}.raptr.preprocess -r ../../reference/umd3_kary_nmask_hgap.fa -t 10 -p ../../tmp/; done

```

## TFBS preparation

I want to see the impact of variants on TFBSs, so I better do the liftover from btau4 to umd3. Hopefully, I maintain at least 90% of the sites!

Liftover should be pretty easy to manage at this point. Here's the

> Blade14:  /mnt/iscsi/vnx_gliu_7/ruminant_project

```bash
/home/dbickhart/bin/liftOver cattle_bc_tfbs_btau4.bed bosTau4ToBosTau6.over.chain cattle_bc_tfbs_umd3_liftover.bed cattle_bc_tfbs_umd3_liftover.unmapped 

wc -l cattle_bc_tfbs_btau4.bed cattle_bc_tfbs_umd3_liftover.bed cattle_bc_tfbs_umd3_liftover.unmapped
  379333 cattle_bc_tfbs_btau4.bed
  377281 cattle_bc_tfbs_umd3_liftover.bed
    4104 cattle_bc_tfbs_umd3_liftover.unmapped

# Well! That's better than I hoped!
# Just taking a brief look at the unmapped coordinates quickly
grep '#Deleted' cattle_bc_tfbs_umd3_liftover.unmapped | wc -l
	1676
grep '#Partially' cattle_bc_tfbs_umd3_liftover.unmapped | wc -l
	349

# OK, I didn't want those sites in any case, it looks like!
```

## MGE identification

Looking through my perl scripts, it looks like I have a script that automatically parses transchr ID'ed discordant reads for repeat identification. I just need to set up the proper files and let it run!

I'm going to prepare the bed files before I start running anything, just so that I have my ducks in a row.

> Blade14: /mnt/iscsi/vnx_gliu_7/ruminant_project/gene_data

```bash
wget http://hgdownload.soe.ucsc.edu/goldenPath/bosTau6/database/rmsk.txt.gz
wget http://hgdownload.soe.ucsc.edu/goldenPath/bosTau6/database/ensGene.txt.gz

gunzip *.gz

# Now to convert them to bed format and to extract the upstream regions of the genes
perl -lane 'if($F[3] eq "-"){$e = $F[5] + 2000; print "$F[2]\t$F[5]\t$e\t$F[12]";}else{$s = $F[4] - 2000; if($s < 0){$s = 0;} print "$F[2]\t$s\t$F[4]\t$F[12]";}' < ensGene.txt > umd3_ensgene_2kb_upstream.bed
perl -lane 'print "$F[2]\t$F[4]\t$F[5]\t$F[12]";' < ensGene.txt > umd3_ensgene.bed

# Now for repeatmasker
perl -lane 'print "$F[5]\t$F[6]\t$F[7]\t$F[10]";' < rmsk.txt > umd3_repeatmask_named.bed
```

I have the gene bed files that I need, let's grep out the transchr divet reads so that I can use them in the pipeline script.

> Blade14: /mnt/iscsi/vnx_gliu_7/ruminant_project/goat_buff_bams

```bash
# Testing on AG280
grep 'transchr' AG280.raptr.preprocess.D.divet > AG280.raptr.preprocess.D.transchr.divet
perl ~/perl_toolchain/sequence_data_scripts/transchrRepeatID.pl -d AG280.raptr.preprocess.D.transchr.divet -f ../gene_data/umd3_repeatmask_named.bed -o AG280.transchr.repeat.mge.bed

# WHOOPS! Mistake! Gotta convert the transchr file to bedpe!
# Also, this ID script ensures that the transchr alignments are not ambiguous
perl ~/perl_toolchain/sequence_data_scripts/transchrIdentificationAlgorithm.pl -d AG280.raptr.preprocess.D.divet -o AG280.raptr.preprocess.D.tranchr.bedpe

# Queuing these up to run them faster!
for i in AG302 AG304 AG306 ITWB10 ITWB11 ITWB12 ITWB13; do divet=${i}.raptr.preprocess.D.divet; echo $divet; output=${i}.raptr.preprocess.D.tranchr.bedpe; perl ~/perl_toolchain/sequence_data_scripts/transchrIdentificationAlgorithm.pl -d $divet -o $output &
done

# OK, trying out one of the goat samples
perl ~/perl_toolchain/sequence_data_scripts/transchrRepeatID.pl -d AG280.raptr.preprocess.D.tranchr.bedpe -f ../gene_data/umd3_repeatmask_named.bed -o AG280.transchr.repeat.mge.bed

wc -l AG280.transchr.repeat.mge.bed
	463 AG280.transchr.repeat.mge.bed

# Not many! Let's see how this pans out with our gene intersection
intersectBed -a AG280.transchr.repeat.mge.bed -b ../gene_data/umd3_ensgene_2kb_upstream.bed -wb | cut -f13 | uniq
ENSBTAG00000047403.1
ENSBTAG00000011541.5
ENSBTAG00000035268.1
ENSBTAG00000047122.1
ENSBTAG00000014435.3
ENSBTAG00000045518.1

# OK! Something to work with! Let's get the others queued up

# I wrote a Java program that should streamline this. Let's test it out
~/jdk1.8.0_05/bin/java -jar ~/MEIDivetID/store/MEIDivetID.jar -i AG280.raptr.preprocess.D.divet -r ../gene_data/umd3_repeatmask_named.bed -o AG280.raptr.MEIDivet

# I need to bugfix it, unfortunately!

# Going to change the repeat bed and condense down the repeat family names
perl -lane '$e = substr($F[3], 0, 4); print "$F[0]\t$F[1]\t$F[2]\t$e";' < umd3_repeatmask_named.bed > umd3_repeatmask_named.reduced.bed

~/jdk1.8.0_05/bin/java -jar ~/MEIDivetID/store/MEIDivetID.jar -i AG280.raptr.preprocess.D.divet -r ../gene_data/umd3_repeatmask_named.reduced.bed -o AG280.raptr.MEIDivet

intersectBed -a AG280.raptr.MEIDivet.putative.mei -b ../gene_data/umd3_ensgene_2kb_upstream.bed -wb | cut -f13 | uniq | wc -l
1865

# OK, there are more, but it's not every gene. about 10%
for i in *.divet; do name=`echo $i | cut -d'.' -f1`; echo $name; out=${name}.raptr.MEIDivet; ~/jdk1.8.0_05/bin/java -jar ~/MEIDivetID/store/MEIDivetID.jar -i $i -r ../gene_data/umd3_repeatmask_named.reduced.bed -o $out; done

wc -l *putative.mei
    94748 AG280.raptr.MEIDivet.putative.mei
    92157 AG302.raptr.MEIDivet.putative.mei
   175003 AG304.raptr.MEIDivet.putative.mei
    97134 AG306.raptr.MEIDivet.putative.mei
   457612 ITWB10.raptr.MEIDivet.putative.mei
   584112 ITWB11.raptr.MEIDivet.putative.mei
   843911 ITWB12.raptr.MEIDivet.putative.mei
   539155 ITWB13.raptr.MEIDivet.putative.mei
   646520 ITWB14.raptr.MEIDivet.putative.mei
   328120 ITWB15.raptr.MEIDivet.putative.mei
        0 ITWB1.raptr.MEIDivet.putative.mei
   147413 ITWB2.raptr.MEIDivet.putative.mei
   130187 ITWB3.raptr.MEIDivet.putative.mei
   134062 ITWB4.raptr.MEIDivet.putative.mei
   106754 ITWB5.raptr.MEIDivet.putative.mei
    84045 ITWB6.raptr.MEIDivet.putative.mei
   118577 ITWB7.raptr.MEIDivet.putative.mei
    67095 ITWB9.raptr.MEIDivet.putative.mei
   247940 PC1.raptr.MEIDivet.putative.mei

# Let's associate them all with upstream regions
# I'm noticing that the single base repeats are way too prevalent in some of the waterbuffalo. 
# Removing single base repeats from the analysis
for i in mei_calls/*putative.mei; do name=`echo $i | cut -d'.' -f1`; echo $name; perl -lane 'if($F[3] =~ /^\([TACG].+/){next;}else{print $_;}' < $i > $name.raptr.MEIDivet.filtered.putative.mei; done

# That reduced the problem a bit, and those calls are more likely misalignments

for i in mei_calls/*filtered.putative.mei; do name=`basename $i | cut -d'.' -f1`; echo $name; intersectBed -a $i -b ../gene_data/umd3_ensgene_2kb_upstream.bed -wb | uniq > mei_calls/${name}.mei.gene.2kb_upstream.bed; done

# In order to find the common elements, I'm going to push the limits of my namelistvenn script
for i in mei_calls/*mei.gene.2kb_upstream.bed; do name=`basename $i | cut -d'.' -f1`; echo $name; perl -lane 'print $F[-1];' < $i | uniq > mei_calls/${name}.mei.gene.2kb_upstream.list; done
for i in mei_calls/I*.mei.gene.2kb_upstream.list; do echo -n "$i "; done; echo
perl ~/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl mei_calls/ITWB10.mei.gene.2kb_upstream.list mei_calls/ITWB11.mei.gene.2kb_upstream.list mei_calls/ITWB12.mei.gene.2kb_upstream.list mei_calls/ITWB13.mei.gene.2kb_upstream.list mei_calls/ITWB14.mei.gene.2kb_upstream.list mei_calls/ITWB15.mei.gene.2kb_upstream.list mei_calls/ITWB2.mei.gene.2kb_upstream.list mei_calls/ITWB3.mei.gene.2kb_upstream.list mei_calls/ITWB4.mei.gene.2kb_upstream.list mei_calls/ITWB5.mei.gene.2kb_upstream.list mei_calls/ITWB6.mei.gene.2kb_upstream.list mei_calls/ITWB7.mei.gene.2kb_upstream.list mei_calls/ITWB9.mei.gene.2kb_upstream.list mei_calls/PC1.mei.gene.2kb_upstream.list > intersections.out
perl -lane '($b)= $F[0] =~ tr/;/;/; if($b >= 12){print $_;}' < intersections.out
	1;2;3;4;5;6;7;8;10;11;12;13;14  1
	1;2;3;4;5;6;7;8;9;10;11;12;13   1
	1;2;3;4;5;6;7;8;9;10;11;12;13;14        36
	1;2;3;4;5;6;7;8;9;10;11;12;14   265
	1;2;3;4;5;6;7;8;9;10;11;13;14   2
	1;2;3;4;5;6;7;8;9;10;12;13;14   2
	1;2;3;4;5;6;8;9;10;11;12;13;14  1
	1;2;3;4;6;7;8;9;10;11;12;13;14  2
	1;2;3;5;6;7;8;9;10;11;12;13;14  1
	1;2;4;5;6;7;8;9;10;11;12;13;14  1

# OK! Not a bad result! Let's find out which genes are in these shared groups though!
perl ~/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl -o mei_calls/ITWB10.mei.gene.2kb_upstream.list mei_calls/ITWB11.mei.gene.2kb_upstream.list mei_calls/ITWB12.mei.gene.2kb_upstream.list mei_calls/ITWB13.mei.gene.2kb_upstream.list mei_calls/ITWB14.mei.gene.2kb_upstream.list mei_calls/ITWB15.mei.gene.2kb_upstream.list mei_calls/ITWB2.mei.gene.2kb_upstream.list mei_calls/ITWB3.mei.gene.2kb_upstream.list mei_calls/ITWB4.mei.gene.2kb_upstream.list mei_calls/ITWB5.mei.gene.2kb_upstream.list mei_calls/ITWB6.mei.gene.2kb_upstream.list mei_calls/ITWB7.mei.gene.2kb_upstream.list mei_calls/ITWB9.mei.gene.2kb_upstream.list mei_calls/PC1.mei.gene.2kb_upstream.list > intersections.out

for i in `ls group_*.txt | perl -lane '($b)= $F[0] =~ tr/_//; if($b >= 13){print $_;}'`; do echo $i; mv $i mei_calls/buffalo.$i; done

# Let's do the same for goat
for i in mei_calls/A*mei.gene.2kb_upstream.list; do echo -n "$i "; done; echo
perl ~/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl mei_calls/AG280.mei.gene.2kb_upstream.list mei_calls/AG302.mei.gene.2kb_upstream.list mei_calls/AG304.mei.gene.2kb_upstream.list mei_calls/AG306.mei.gene.2kb_upstream.list
File Number 1: mei_calls/AG280.mei.gene.2kb_upstream.list
File Number 2: mei_calls/AG302.mei.gene.2kb_upstream.list
File Number 3: mei_calls/AG304.mei.gene.2kb_upstream.list
File Number 4: mei_calls/AG306.mei.gene.2kb_upstream.list
Set     Count
1       695
1;2     69
1;2;3   64
1;2;3;4 158
1;2;4   41
1;3     190
1;3;4   135
1;4     130
2       566
2;3     152
2;3;4   83
2;4     84
3       1271
3;4     222
4       741

perl ~/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl -o mei_calls/AG280.mei.gene.2kb_upstream.list mei_calls/AG302.mei.gene.2kb_upstream.list mei_calls/AG304.mei.gene.2kb_upstream.list mei_calls/AG306.mei.gene.2kb_upstream.list
mv group_1_2_3_4.txt mei_calls/goat.group_1_2_3_4.txt

# we'll use 4 groups for Goat and 13+ for Buffalo
# So 158 genes for Goat
# 312 in Buffalo
cat mei_calls/buffalo.group_1_2_*.txt | sort | uniq > mei_calls/buffalo.group_13_plus.txt
wc -l mei_calls/buffalo.group_13_plus.txt mei_calls/goat.group_1_2_3_4.txt
 312 mei_calls/buffalo.group_13_plus.txt
 158 mei_calls/goat.group_1_2_3_4.txt
 470 total

# Let's see the overlap
perl ~/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl mei_calls/buffalo.group_13_plus.txt mei_calls/goat.group_1_2_3_4.txt
File Number 1: mei_calls/buffalo.group_13_plus.txt
File Number 2: mei_calls/goat.group_1_2_3_4.txt
Set     Count
1       284
1;2     28
2       130

# Interesting! So about 28 genes shared!
perl ~/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl -o mei_calls/buffalo.group_13_plus.txt mei_calls/goat.group_1_2_3_4.txt
mv group_1_2.txt mei_calls/goat.buff.mei.shared.txt
# 8 of the combined set genes did not map to the DAVID database
```

#### David db on shared goat buff MEI gene lists
>ENSBTAG00000019788	TEA domain family member 4	Related Genes	Bos taurus
>GOTERM_BP_FAT	regulation of transcription, DNA-dependent, regulation of transcription, regulation of RNA metabolic process,
>GOTERM_MF_FAT	DNA binding, transcription factor activity, transcription regulator activity,
ENSBTAG00000033412	beta 1,3-galactosyltransferase-like	Related Genes	Bos taurus
GOTERM_BP_FAT	intracellular protein transport, protein localization, protein transport, vesicle-mediated transport, cellular protein localization, establishment of protein localization, intracellular transport, cellular macromolecule localization,
GOTERM_CC_FAT	endoplasmic reticulum, membrane coat, coated membrane,
INTERPRO	Clathrin/coatomer adaptor, adaptin-like, N-terminal, Fringe-like, Armadillo-like helical,
ENSBTAG00000030297	hepatitis A virus cellular receptor 1 N-terminal domain containing protein	Related Genes	Bos taurus
INTERPRO	Immunoglobulin subtype, Immunoglobulin-like, Immunoglobulin V-set, Immunoglobulin-like fold,
SMART	IG,
SP_PIR_KEYWORDS	receptor,
ENSBTAG00000013866	ribosomal protein S27 pseudogene	Related Genes	Bos taurus
GOTERM_BP_FAT	translation,
GOTERM_CC_FAT	ribosome, ribonucleoprotein complex, non-membrane-bounded organelle, intracellular non-membrane-bounded organelle,
GOTERM_MF_FAT	structural constituent of ribosome, structural molecule activity,
ENSBTAG00000035027	similar to MOR 3Beta4	Related Genes	Bos taurus
GOTERM_BP_FAT	cell surface receptor linked signal transduction, G-protein coupled receptor protein signaling pathway,
GOTERM_CC_FAT	integral to membrane, intrinsic to membrane,
GOTERM_MF_FAT	olfactory receptor activity,
KEGG_PATHWAY	Olfactory transduction,
ENSBTAG00000009542	zinc finger CCCH-type containing 10	Related Genes	Bos taurus
GOTERM_MF_FAT	zinc ion binding, ion binding, cation binding, metal ion binding, transition metal ion binding,
INTERPRO	Zinc finger, CCCH-type,
SMART	ZnF_C3H1,
SP_PIR_KEYWORDS	metal-binding, zinc,



## SNP and INDEL ID

Let's start calling SNPs and indels for all of the bams I have on Goat and Buffalo:

> Blade14: /mnt/iscsi/vnx_gliu_7/ruminant_project/goat_buff_bams

```bash
# For Goat
perl ~/perl_toolchain/sequence_data_scripts/samtoolsSNPFork.pl -r ../../reference/umd3_kary_unmask_ngap.fa -i ./AG280/AG280.1.nodup.bam,./AG302/AG302.1.nodup.bam,./AG304/AG304.1.nodup.bam,./AG306/AG306.1.nodup.bam -o goat_umd3_comparative_snp_calling.vcf -n 5 -s ../../reference/samtools_chr_segs.txt -t 1

# For Buffalo
perl ~/perl_toolchain/sequence_data_scripts/samtoolsSNPFork.pl -r ../../reference/umd3_kary_unmask_ngap.fa -i ./ITWB10/ITWB10.merged.bam,./ITWB11/ITWB11.merged.bam,./ITWB12/ITWB12.merged.bam,./ITWB13/ITWB13.merged.bam,./ITWB14/ITWB14.merged.bam,./ITWB15/ITWB15.merged.bam,./ITWB1/ITWB1.merged.bam,./ITWB2/ITWB2.merged.bam,./ITWB3/ITWB3.merged.bam,./ITWB4/ITWB4.merged.bam,./ITWB5/ITWB5.merged.bam,./ITWB6/ITWB6.merged.bam,./ITWB7/ITWB7.merged.bam,./ITWB9/ITWB9.merged.bam,./PC1/PC1.merged.bam -o buffalo_umd3_comparative_snp_calling.vcf -n 5 -s ../../reference/samtools_chr_segs.txt -t 1

# Removing the background snps and indels using a BTHO+BTJE vcf
intersectBed -a goat_umd3_comparative_snp_calling.vcf.samtools.filtered.vcf -b /mnt/iscsi/vnx_gliu_7/100_base_run/samtools_snps/btho_btje_samples.samtools.filtered.vcf -v > goat_umd3_comparative_snp_calling.vcf.samtools.nobackground.vcf

intersectBed -a buffalo_umd3_comparative_snp_calling.vcf.samtools.filtered.vcf -b /mnt/iscsi/vnx_gliu_7/100_base_run/samtools_snps/btho_btje_samples.samtools.filtered.vcf -v > buffalo_umd3_comparative_snp_calling.vcf.samtools.nobackground.vcf

# Getting only the homozygous snps/indels
perl -lane 'if($_ =~ /^#/){print $_;}else{$not = 0; for($x = 9; $x < scalar(@F); $x++){@b = split(/:/, $F[$x]); if($b[0] ne "1/1"){$not = 1; last;}} if(!$not){print $_;}}' < goat_umd3_comparative_snp_calling.vcf.samtools.nobackground.vcf > goat_umd3_comparative_snp_calling.vcf.samtools.nobackground.hom.vcf

perl -lane 'if($_ =~ /^#/){print $_;}else{$not = 0; for($x = 9; $x < scalar(@F); $x++){@b = split(/:/, $F[$x]); if($b[0] ne "1/1"){$not = 1; last;}} if(!$not){print $_;}}' < buffalo_umd3_comparative_snp_calling.vcf.samtools.nobackground.vcf > buffalo_umd3_comparative_snp_calling.vcf.samtools.nobackground.hom.vcf

wc -l *.hom.vcf
       191 buffalo_umd3_comparative_snp_calling.vcf.samtools.nobackground.hom.vcf
   7607135 goat_umd3_comparative_snp_calling.vcf.samtools.nobackground.hom.vcf
   7607326 total

# Hmmm, I'm missing allot of buffalo here
# Going to try a slightly more permissive setting
perl -lane 'if($_ =~ /^#/){print $_;}else{$not = 0; for($x = 9; $x < scalar(@F); $x++){@b = split(/:/, $F[$x]); if($b[0] ne "1/1" && $b[0] ne "./."){$not = 1; last;}} if(!$not){print $_;}}' < buffalo_umd3_comparative_snp_calling.vcf.samtools.nobackground.vcf > buffalo_umd3_comparative_snp_calling.vcf.samtools.nobackground.hom.perm.vcf

# Nope! Too permissive. Trying to stop at 3 ambiguous sites
perl -lane 'if($_ =~ /^#/){print $_;}else{$not = 0; for($x = 9; $x < scalar(@F); $x++){@b = split(/:/, $F[$x]); $discrep = 0; if($b[0] ne "1/1"){if($b[0] eq "./."){$discrep++; if($discrep > 2){$not = 1; last;}else{next;}}$not = 1; last;} } if(!$not){print $_;}}' < buffalo_umd3_comparative_snp_calling.vcf.samtools.nobackground.vcf > buffalo_umd3_comparative_snp_calling.vcf.samtools.nobackground.hom.perm.vcf

# OK, now trying to get only the snps that intersect with gene upstream regions
grep -v INDEL goat_umd3_comparative_snp_calling.vcf.samtools.nobackground.hom.vcf | intersectBed -a stdin -b ../gene_data/umd3_ensgene_2kb_upstream.bed -wb > goat_umd3_comparative_snp.hom.2kb_upstream.vcf
grep -v INDEL buffalo_umd3_comparative_snp_calling.vcf.samtools.nobackground.hom.perm.vcf | intersectBed -a stdin -b ../gene_data/umd3_ensgene_2kb_upstream.bed -wb > buffalo_umd3_comparative_snp.hom.2kb_upstream.vcf

wc -l *upstream.vcf
   994171 buffalo_umd3_comparative_snp.hom.2kb_upstream.vcf
   150644 goat_umd3_comparative_snp.hom.2kb_upstream.vcf
  1144815 total

# Now, let's see how much of the conserved TFBS are intersecting the snp
perl -lane '$e = $F[1] + 1; print "$F[0]\t$F[1]\t$e\t$F[-1]";' <  buffalo_umd3_comparative_snp.hom.2kb_upstream.vcf > buffalo_umd3_comparative_snp.hom.2kb_upstream.bed
perl -lane '$e = $F[1] + 1; print "$F[0]\t$F[1]\t$e\t$F[-1]";' < goat_umd3_comparative_snp.hom.2kb_upstream.vcf > goat_umd3_comparative_snp.hom.2kb_upstream.bed

intersectBed -a buffalo_umd3_comparative_snp.hom.2kb_upstream.bed -b ../cattle_bc_tfbs_umd3_liftover.bed | wc -l
26038
intersectBed -a goat_umd3_comparative_snp.hom.2kb_upstream.bed -b ../cattle_bc_tfbs_umd3_liftover.bed | wc -l
4437

intersectBed -a buffalo_umd3_comparative_snp.hom.2kb_upstream.bed -b ../cattle_bc_tfbs_umd3_liftover.bed > buffalo_umd3_tfbs_intersect.bed
intersectBed -a goat_umd3_comparative_snp.hom.2kb_upstream.bed -b ../cattle_bc_tfbs_umd3_liftover.bed > goat_umd3_tfbs_intersect.bed

# Note, alot of these are duplicates from the TFBS bed
cat goat_umd3_tfbs_intersect.bed | cut -f4 | uniq > goat_umd3_tfbs_intersect.list
cat buffalo_umd3_tfbs_intersect.bed | cut -f4 | uniq > buffalo_umd3_tfbs_intersect.list

perl ~/perl_toolchain/bed_cnv_fig_table_pipeline/nameListVennCount.pl -o goat_umd3_tfbs_intersect.list buffalo_umd3_tfbs_intersect.list
	File Number 1: goat_umd3_tfbs_intersect.list
	File Number 2: buffalo_umd3_tfbs_intersect.list
	Set     Count
	1       227
	1;2     974
	2       3803

	Group: 1 in output file: group_1.txt
	Group: 1;2 in output file: group_1_2.txt
	Group: 2 in output file: group_2.txt

mv group_1_2.txt goat_buffal_umd3_tfbs_intersect.snp.share.list
mv group_2.txt buffalo_umd3_tfbs_intersect.snp.uniq.list
mv group_1.txt goat_umd3_tfbs_intersect.snp.uniq.list
```

## Getting segmental duplication regions

I am going to get the seg dups really quickly and filter away potentially spurious calls quickly.

> Blade14: /mnt/iscsi/vnx_gliu_7/ruminant_project/goat_buff_bams/jarms

```bash
# Getting dups
for i in *.bed; do name=`echo $i | cut -d'.' -f1`; perl -e 'chomp(@ARGV); open(IN, "< $ARGV[0]"); while(<IN>){chomp; @s = split(/\t/); if($s[5] < 0.05 && $s[6] < 0.05 && $s[3] eq "duplication"){print "$s[0]\t$s[1]\t$s[2]\t$ARGV[1]\t$s[4]\n";}}' $i $name > $name.filtered.dups.bed; done
for i in *.bed; do name=`echo $i | cut -d'.' -f1`; perl -e 'chomp(@ARGV); open(IN, "< $ARGV[0]"); while(<IN>){chomp; @s = split(/\t/); if($s[5] < 0.05 && $s[6] < 0.05 && $s[3] eq "deletion"){print "$s[0]\t$s[1]\t$s[2]\t$ARGV[1]\t$s[4]\n";}}' $i $name > $name.filtered.dels.bed; done
wc -l *.dups.bed
   3144 AG280.filtered.dups.bed
   3481 AG304.filtered.dups.bed
   2810 AG306.filtered.dups.bed
   3014 ITWB10.filtered.dups.bed
   3089 ITWB11.filtered.dups.bed
   3244 ITWB12.filtered.dups.bed
   2812 ITWB13.filtered.dups.bed
   2539 ITWB14.filtered.dups.bed
   3260 ITWB15.filtered.dups.bed
   3303 ITWB1.filtered.dups.bed
   3394 ITWB2.filtered.dups.bed
   3507 ITWB3.filtered.dups.bed
   3302 ITWB4.filtered.dups.bed
   3165 ITWB5.filtered.dups.bed
   2128 ITWB6.filtered.dups.bed
   3328 ITWB7.filtered.dups.bed
   2975 PC1.filtered.dups.bed

wc -l *.dels.bed
     71 AG280.filtered.dels.bed
     50 AG304.filtered.dels.bed
    109 AG306.filtered.dels.bed
   1878 ITWB10.filtered.dels.bed
   1796 ITWB11.filtered.dels.bed
   1580 ITWB12.filtered.dels.bed
   1783 ITWB13.filtered.dels.bed
   1384 ITWB14.filtered.dels.bed
   1934 ITWB15.filtered.dels.bed
   2124 ITWB1.filtered.dels.bed
   2307 ITWB2.filtered.dels.bed
   2191 ITWB3.filtered.dels.bed
   2250 ITWB4.filtered.dels.bed
   2161 ITWB5.filtered.dels.bed
   1466 ITWB6.filtered.dels.bed
   2275 ITWB7.filtered.dels.bed
   1926 PC1.filtered.dels.bed
  27285 total

# Interesting thing on the deletions in the goat! Let's condense down to seg dups
# Noticed some discrepencies in the data. Summarizing below
ls ITW*.dups.bed > ITWB_dup_file.list
echo -e "/mnt/iscsi/vnx_gliu_7/ruminant_project/gene_data/umd3_ensgene.bed\tensgene" > genedb.list
~/jdk1.8.0_05/bin/java -jar ~/AnnotateUsingGenomicInfo/store/AnnotateUsingGenomicInfo.jar -d genedb.list -i ITWB_dup_file.list -o buffalo_segdups -t
perl -lane 'if($F[4] >= 12){print "$F[1]\t$F[2]\t$F[3]\tbuffalo";}' < buffalo_segdups_regions.tab > buffalo_fixed_segdups.bed

```

| Group | Type | Count | Total Len |
| :--- | :--- | ---: | ---: |
Buffalo	| Dups	| 14,123 | 24,099,877
Buffalo | Dels | 4,982 | 50,768,518
Goat | Dups | 1,730 | 2,443,230,270
Goat | Dels | 145 | 6,534,855

I think that the Goat has way too many mapping issues to be reliable. Let's check this out.

#### X coverage

| Group | mapped X cov | unmapped X cov |
| :--- | :--- | :--- |
Goat | 0.9 - 2.3 | 0.8 - 2.1 |
Buffalo | 0.9 - 12.6 | 0.4 - 12.3

Let's stick with buffalo for Seg dups.

> Blade14: /mnt/iscsi/vnx_gliu_7/100_base_run/jarms

```bash
for i in *.bed; do name=`echo $i | cut -d'.' -f1`; perl -e 'chomp(@ARGV); open(IN, "< $ARGV[0]"); while(<IN>){chomp; @s = split(/\t/); if($s[5] < 0.05 && $s[6] < 0.05 && $s[3] eq "duplication"){print "$s[0]\t$s[1]\t$s[2]\t$ARGV[1]\t$s[4]\n";}}' $i $name > $name.filtered.dups.bed; done
for i in *.bed; do name=`echo $i | cut -d'.' -f1`; perl -e 'chomp(@ARGV); open(IN, "< $ARGV[0]"); while(<IN>){chomp; @s = split(/\t/); if($s[5] < 0.05 && $s[6] < 0.05 && $s[3] eq "deletion"){print "$s[0]\t$s[1]\t$s[2]\t$ARGV[1]\t$s[4]\n";}}' $i $name > $name.filtered.dels.bed; done

for i in BTHO*.filtered.dups.bed; do name=`echo $i | cut -d'.' -f1`; echo -e "$i\t$name"; done > cattle_background_dup.list
for i in BTJE*.filtered.dups.bed; do name=`echo $i | cut -d'.' -f1`; echo -e "$i\t$name"; done >> cattle_background_dup.list

~/jdk1.8.0_05/bin/java -jar ~/AnnotateUsingGenomicInfo/store/AnnotateUsingGenomicInfo.jar -d /mnt/iscsi/vnx_gliu_7/ruminant_project/goat_buff_bams/jarms/genedb.list -i cattle_background_dup.list -o cattle_background -t
perl -lane 'if($F[4] >= 19){print "$F[1]\t$F[2]\t$F[3]\tcattle";}' < cattle_background_regions.tab > cattle_background_fixed_segdups.bed

# Now, removing the background to get out the segdups
wc -l /mnt/iscsi/vnx_gliu_7/ruminant_project/goat_buff_bams/jarms/buffalo_fixed_segdups.bed cattle_background_fixed_segdups.bed
 130 /mnt/iscsi/vnx_gliu_7/ruminant_project/goat_buff_bams/jarms/buffalo_fixed_segdups.bed
  92 cattle_background_fixed_segdups.bed
 222 total

intersectBed -a /mnt/iscsi/vnx_gliu_7/ruminant_project/goat_buff_bams/jarms/buffalo_fixed_segdups.bed -b cattle_background_fixed_segdups.bed -v | wc -l
101
intersectBed -a /mnt/iscsi/vnx_gliu_7/ruminant_project/goat_buff_bams/jarms/buffalo_fixed_segdups.bed -b cattle_background_fixed_segdups.bed -v | perl ~/perl_toolchain/bed_cnv_fig_table_pipeline/bed_length_sum.pl
        Interval Numbers:       101
        Total Length:           276399
        Length Average:         2736.62376237624
        Length Median:          999
        Length Stdev:           7550.02919783416
        Smallest Length:        999
        Largest Length:         71499

intersectBed -a /mnt/iscsi/vnx_gliu_7/ruminant_project/goat_buff_bams/jarms/buffalo_fixed_segdups.bed -b cattle_background_fixed_segdups.bed -v | perl -lane 'if($F[2] - $F[1] > 5000){print $_;}' | bed_length_sum.pl
        Interval Numbers:       11
        Total Length:           176,989

# And the shared seg dups

```

I've found a discrepency in my detection of deletions. I want to try to clear it up a bit and make the figures more accurate. 

> Blade14: /mnt/iscsi/vnx_gliu_7/100_base_run/jarms

```bash
for i in BTHO*.filtered.dels.bed; do name=`echo $i | cut -d'.' -f1`; echo -e "$i\t$name"; done > cattle_background_del.list
for i in BTJE*.filtered.dels.bed; do name=`echo $i | cut -d'.' -f1`; echo -e "$i\t$name"; done >> cattle_background_del.list

~/jdk1.8.0_05/bin/java -jar ~/AnnotateUsingGenomicInfo/store/AnnotateUsingGenomicInfo.jar -d /mnt/iscsi/vnx_gliu_7/ruminant_project/goat_buff_bams/jarms/genedb.list -i cattle_background_del.list -o cattle_background_dels -t

perl -e '$h = <>; while(<>){chomp; @s = split(/\t/); print "$s[1]\t$s[2]\t$s[3]\tcattle\n";}' < cattle_background_dels_regions.tab > cattle_background_dels_cnvrs.bed
perl -e '$h = <>; while(<>){chomp; @s = split(/\t/); print "$s[1]\t$s[2]\t$s[3]\tbuffalo\n";}' < /mnt/iscsi/vnx_gliu_7/ruminant_project/goat_buff_bams/jarms/buffalo_deletions_regions.tab > /mnt/iscsi/vnx_gliu_7/ruminant_project/goat_buff_bams/jarms/buffalo_deletions_regions.dels.bed

intersectBed -a /mnt/iscsi/vnx_gliu_7/ruminant_project/goat_buff_bams/jarms/buffalo_deletions_regions.dels.bed -b cattle_background_dels_cnvrs.bed -v | intersectBed -a /mnt/iscsi/vnx_gliu_7/ruminant_project/gene_data/umd3_ensgene.bed -b stdin | wc -l
854

# That's still a large amount... let's see the regions that are deleted in all Buffalo
# Err, only 8 regions! Let's do 50% frequency
perl -e '$h = <>; while(<>){chomp; @s = split(/\t/); if($s[4] >= 7){print "$s[1]\t$s[2]\t$s[3]\tbuffalo\n";}}' < /mnt/iscsi/vnx_gliu_7/ruminant_project/goat_buff_bams/jarms/buffalo_deletions_regions.tab > /mnt/iscsi/vnx_gliu_7/ruminant_project/goat_buff_bams/jarms/buffalo_deletions_regions.gt50perc.bed

intersectBed -a /mnt/iscsi/vnx_gliu_7/ruminant_project/goat_buff_bams/jarms/buffalo_deletions_regions.gt50perc.bed -b cattle_background_dels_cnvrs.bed -v | intersectBed -a /mnt/iscsi/vnx_gliu_7/ruminant_project/gene_data/umd3_ensgene.bed -b stdin | uniq | wc -l
102

intersectBed -a /mnt/iscsi/vnx_gliu_7/ruminant_project/goat_buff_bams/jarms/buffalo_deletions_regions.gt50perc.bed -b cattle_background_dels_cnvrs.bed -v | intersectBed -a /mnt/iscsi/vnx_gliu_7/ruminant_project/gene_data/umd3_ensgene.bed -b stdin | perl -lane '$F[3] =~ s/(ENSBTAG.+)\.\d+/$1/; print $F[3];' | uniq | wc -l
94 <- unique genes

# Checking the "levels" from Jarms
for i in /mnt/iscsi/vnx_gliu_7/ruminant_project/goat_buff_bams/jarms/ITWB*jarmscnvs.bed.levels; do name=`basename $i | cut -d'.' -f1`; perl -e 'chomp(@ARGV); open(IN, "< $ARGV[0]"); while(<IN>){chomp; @F = split(/\t/); if($F[3] < 0.5){print "$F[0]\t$F[1]\t$F[2]\t$ARGV[1]\n";}} close IN;' $i $name; done | perl ~/bin/sortBedFileSTDIN.pl | mergeBed -i stdin -nms | perl -lane 'my %h; my @s = split(/;/, $F[3]); foreach $k (@s){$h{$k} = 1;} if(scalar(keys(%h)) >= 13){print join("\t", @F);}' > /mnt/iscsi/vnx_gliu_7/ruminant_project/goat_buff_bams/jarms/buffalo_jarms_common_del_regions.bed

intersectBed -a /mnt/iscsi/vnx_gliu_7/ruminant_project/goat_buff_bams/jarms/buffalo_jarms_common_del_regions.bed -b cattle_background_dels_cnvrs.bed -v | intersectBed -a /mnt/iscsi/vnx_gliu_7/ruminant_project/gene_data/umd3_ensgene.bed -b stdin -f 0.90 -r | wc -l
7

# 7 genes that are reciprocally deleted! Actually 4 after removing redundant entries
# That makes 5 with the Bitter taste receptor (missing just one animal that had 0.66)
```

