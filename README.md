# Pipeline for genomic analysis of _Braula coeca_

## Genome sequencing and assembly of _Braula coeca_
1. Basecall of raw data using Guppy v5.0.11 and the "sup" algorithm
```bash
Guppy_5_0_11/bin/guppy_basecaller --device auto --input_path fast5_pass/ --recursive --save_path Braula_basecalled --config dna_r9.4.1_450bps_sup.cfg
```
2. Quality assessment of the MinION run with PycoQC
```bash
pycoQC -f sequencing_summary.txt -o sequencing_summary_braula.html
```
3. Hybrid assembly of the genome using MaSuRCA v4.0.3 (with the Cabog assembler)
> The following was changed in the _Braula_ analysis as compared to the default configuration file
```bash
PE= pe 150 50 [list of Illumina Fastq files]
NANOPORE=Braula_18_06_21.fastq
USE_LINKING_MATES = 1
NUM_THREADS = 24
JF_SIZE = 5400000000
FLYE_ASSEMBLY=0
```
4. Polishing of the preliminary assembly with polca as implemented in MaSuRCA
```bash
MaSuRCA-4.0.3/bin/polca.sh -a Braula_masurca403.fasta -r [list of Illumina Fastq files] -t 8
```
5. Estimation of the completeness of the assembly with Busco v5.0.0
```bash
busco -i  Braula_masurca403.fasta.PolcaCorrected.fa -o Braula_Masurca_Busco -l diptera_odb10 -m geno -c 2
```
> Note: Braula_masurca403.fasta.PolcaCorrected.fa was then renamed Braula_assembly_21_09_12.fasta

## Estimation of genome size and endosymbionts detection
1. K-mers frequencies within short-read data obtained using KMC 3
```bash
mkdir tmp
ls *.fastq.gz > FILES
kmc -k21 -t16 -m64 -ci1 -cs10000 @FILES kmcdb tmp
kmc_tools transform kmcdb histogram kmcdb_k21.hist -cx10000
```
2. Genome size and ploidy inferred with GenomeScope v2.0 (with k-mer size = 21 and Smudgeplot)
```bash
Rscript genomescope.R kmcdb_k21.hist <k-mer_length> <read_length> <output_dir> [kmer_max] [verbose]
L=$(smudgeplot.py cutoff kmcdb_k21.hist L)
U=$(smudgeplot.py cutoff kmcdb_k21.hist U)
kmc_tools transform kmcdb -ci"$L" -cx"$U" dump -s kmcdb_L"$L"_U"$U".dump
smudgeplot.py hetkmers -o kmcdb_L"$L"_U"$U" < kmcdb_L"$L"_U"$U".dump
smudgeplot.py plot kmcdb_L"$L"_U"$U"_coverages.tsv
```
3. Contig taxonomy performed using Blobtools (with Diamond as search engine against the Uniprot database using a local copy of the NCBI TaxID file)
* Generating a coverage file using Minimap2 and Blobtools
```bash
minimap2 -ax sr -t 16 [Braula_assembly] [Braula_Illumina_reads] [Braula_SAM_file]
blobtools -i [Braula_assembly] -s [Braula_SAM_file] -o [output Prefix here Braula]
```
* Generating a hits file using Diamond
```bash
diamond blastx --query [Braula_assembly] --db uniprot_ref_proteomes.diamond.dmnd --outfmt 6 --sensitive --max-target-seqs 1 --evalue 1e-25
blobtools taxify -f diamond.out -m uniprot_ref_proteomes.taxids -s 0 -t 2
```
* Creating and viewing blobtools
```bash
blobtools create -i [Braula_assembly] -s [Braula_SAM] -t [taxified Diammond hits] -o [blobplot]
blobtools view -i blobplot.blobDB.json -o [output plots]
blobtools plot -i blobDB.json -o [output plots]
```

## Genome annotation
> Note: Commands are given for _Braula coeca_ but the same commands were conducted to annotate _Leucophenga varia_, _Phortica variegata_ and _Ephydra gracilis_
1. Identification of repeat-enriched regions using RepeatModeler v2.0.1
```bash
RepeatModeler -database Braula -pa 20 -engine ncbi
```
2. First round of gene prediction with Maker
* Options in the script maker_Braula1_opts.ctl to generate the maker_Braula1_opts.ctl file:
```bash
#-----Genome (these are always required)
genome=Braula_assembly_21_09_12.fasta #genome sequence (fasta file or fasta embeded in GFF3 file)
organism_type=eukaryotic #eukaryotic or prokaryotic. Default is eukaryotic

#-----Protein Homology Evidence (for best results provide a file for at least one)
protein=GCF_004354385_Dinn_protein.fasta,GCF_009650485_Dalb_protein.fasta,GCF_018153845_Dbip_protein.fasta,dmel-all-translation-r6.41.fasta,dvir-all-translation-r1.07.fasta  #protein sequence file in fasta format (i.e. from mutiple oransisms)

#-----Repeat Masking (leave values blank to skip repeat masking)
model_org=all #select a model organism for RepBase masking in RepeatMasker
rmlib=Braula-families.fa #provide an organism specific repeat library in fasta format for RepeatMasker
repeat_protein=RepeatPeps.lib #provide a fasta file of transposable element proteins for RepeatRunner
softmask=1 #use soft-masking rather than hard-masking in BLAST (i.e. seg and dust filtering)

#-----Gene Prediction
protein2genome=1 #infer predictions from protein homology, 1 = yes, 0 = no

#-----External Application Behavior Options
cpus=10 #max number of cpus to use in BLAST and RepeatMasker (not for MPI, leave 1 when using MPI)
```
> Notes:
> 
> 1. Two libraries were used to annotate repeats : the "all" library of Repbase ($model_org) and the library specific to _Braula coeca_ generated by RepeatModeler ($rmlib)
> 2. A fasta file of transposable elements proteins was also provided to Maker : RepeatPeps.lib ($repeat_protein)
> 3. option softmask=1
> 4. Five proteomes of related organisms were provided to Maker ($protein) : Drosophila innubila, D. albomicans, D. bipectinata, D. melanogaster and D. virilis ($protein2genome=1)
* Run Maker
```bash
maker -base Braula.round1 -cpus 10 -fix_nucleotide maker_Braula1_opts.ctl maker_bopts.ctl maker_exe.ctl
```
3. First round of training SNAP
* Merge the output files of the round 1 of Maker
```bash
gff3_merge -d Braula.round1_master_datastore_index.log -o Braula.round1.all.maker.gff
```
* Merge the fasta of proteins and transcriptome (not mandatory)
```bash
fasta_merge -d Braula.round1_master_datastore_index.log
```
* Generate mandatory files to train SNAP (genome.ann (ZFF format) and genome.dna (FASTA format))
```bash
maker2zff -n Braula.round1.maker.output/Braula.round1.all.maker.gff
```
* Validate gene models
```bash
fathom genome.ann genome.dna -validate > snap_validate_output.txt
```
* Identify the erroneous model(s)
```bash
cat snap_validate_output.txt | grep "error" 
```
* Remove the gene model with the error:
```bash
grep -vwE "MODEL_ID" genome.ann > genome.ann2
```
* Rerunning fathom should now show no errors:
```bash
fathom genome.ann2 genome.dna -validate 
```
* Generate the remaining necessary files to train SNAP; collect the training sequences and annotations, plus 1000 surrounding bp for training
```bash
fathom genome.ann2 genome.dna -categorize 1000
fathom uni.ann uni.dna -export 1000 -plus           
```
* Create the training parameters
```bash
forge export.ann export.dna
```
* Train SNAP with hmm-assembler
```bash
hmm-assembler.pl Braula.round1 params > Braula.round1.hmm
```
4. Second round of gene prediction with Maker
* Options in the script maker_Braula2_opts.ctl to generate the maker_Braula2_opts.ctl file:
```bash
#-----Genome (these are always required)
genome=Braula_assembly_21_09_12.fasta #genome sequence (fasta file or fasta embeded in GFF3 file)
organism_type=eukaryotic #eukaryotic or prokaryotic. Default is eukaryotic

#-----Re-annotation Using MAKER Derived GFF3
maker_gff= Braula.round1.all.maker.gff #MAKER derived GFF3 file
est_pass=1 #use ESTs in maker_gff: 1 = yes, 0 = no
altest_pass=0 #use alternate organism ESTs in maker_gff: 1 = yes, 0 = no
protein_pass=1 #use protein alignments in maker_gff: 1 = yes, 0 = no
rm_pass=1 #use repeats in maker_gff: 1 = yes, 0 = no

#-----Protein Homology Evidence (for best results provide a file for at least one)
protein=  #protein sequence file in fasta format (i.e. from mutiple oransisms)

#-----Repeat Masking (leave values blank to skip repeat masking)
model_org= #select a model organism for RepBase masking in RepeatMasker
rmlib= #provide an organism specific repeat library in fasta format for RepeatMasker
repeat_protein= #provide a fasta file of transposable element proteins for RepeatRunner
rm_gff= #pre-identified repeat elements from an external GFF3 file
prok_rm=0 #forces MAKER to repeatmask prokaryotes (no reason to change this), 1 = yes, 0 = no
softmask=1 #use soft-masking rather than hard-masking in BLAST (i.e. seg and dust filtering)

#-----Gene Prediction
snaphmm=Braula.round1.hmm #SNAP HMM file
est2genome=0 #infer gene predictions directly from ESTs, 1 = yes, 0 = no
protein2genome=0 #infer predictions from protein homology, 1 = yes, 0 = no

#-----MAKER Behavior Options
pred_stats=1 #report AED and QI statistics for all predictions as well as models
```
* Run Maker
```bash
maker -base Braula.round2 -cpus 15 maker_Braula2_opts.ctl maker_bopts.ctl maker_exe.ctl
```
5. Second round of training SNAP
* Merge the output files of the round 2 of Maker
```bash
gff3_merge -d Braula.round2_master_datastore_index.log -o Braula.round2.all.maker.gff
```
* Merge the fasta of proteins and transcriptome (not mandatory)
```bash
fasta_merge -d Braula.round2_master_datastore_index.log
```
* Generate mandatory files to train SNAP (genome.ann (ZFF format) and genome.dna (FASTA format))
```bash
maker2zff -n Braula.round2.maker.output/Braula.round2.all.maker.gff
```
* Validate gene models
```bash
fathom genome.ann genome.dna -validate > snap_validate_output.txt
```
* Identify the erroneous model(s)
```bash
cat snap_validate_output.txt | grep "error"
```
* Remove the gene model with the error:
```bash
grep -vwE "MODEL_ID" genome.ann > genome.ann2
```
* Rerunning fathom should now show no errors:
```bash
fathom genome.ann2 genome.dna -validate 
```
* Generate the remaining necessary files to train SNAP; collect the training sequences and annotations, plus 1000 surrounding bp for training
```bash
fathom genome.ann2 genome.dna -categorize 1000
fathom uni.ann uni.dna -export 1000 -plus           
```
* Create the training parameters
```bash
forge export.ann export.dna
```
* Train SNAP with hmm-assembler
```bash
hmm-assembler.pl Braula.round2 params > Braula.round2.hmm
```
6. Third round of gene prediction with Maker
* Options in the script maker_Braula3_opts.ctl to generate the maker_Braula3_opts.ctl file
```bash
#-----Genome (these are always required)
genome=Braula_assembly_21_09_12.fasta #genome sequence (fasta file or fasta embeded in GFF3 file)
organism_type=eukaryotic #eukaryotic or prokaryotic. Default is eukaryotic

#-----Re-annotation Using MAKER Derived GFF3
maker_gff= Braula.round2.all.maker.gff #MAKER derived GFF3 file
est_pass=1 #use ESTs in maker_gff: 1 = yes, 0 = no
altest_pass=0 #use alternate organism ESTs in maker_gff: 1 = yes, 0 = no
protein_pass=1 #use protein alignments in maker_gff: 1 = yes, 0 = no
rm_pass=1 #use repeats in maker_gff: 1 = yes, 0 = no

#-----Protein Homology Evidence (for best results provide a file for at least one)
protein=  #protein sequence file in fasta format (i.e. from mutiple oransisms)

#-----Repeat Masking (leave values blank to skip repeat masking)
model_org= #select a model organism for RepBase masking in RepeatMasker
rmlib= #provide an organism specific repeat library in fasta format for RepeatMasker
repeat_protein= #provide a fasta file of transposable element proteins for RepeatRunner
rm_gff= #pre-identified repeat elements from an external GFF3 file
prok_rm=0 #forces MAKER to repeatmask prokaryotes (no reason to change this), 1 = yes, 0 = no
softmask=1 #use soft-masking rather than hard-masking in BLAST (i.e. seg and dust filtering)

#-----Gene Prediction
snaphmm=Braula.round2.hmm #SNAP HMM file
est2genome=0 #infer gene predictions directly from ESTs, 1 = yes, 0 = no
protein2genome=0 #infer predictions from protein homology, 1 = yes, 0 = no

#-----MAKER Behavior Options
pred_stats=1 #report AED and QI statistics for all predictions as well as models
```
* Run Maker
```bash
maker -base Braula.round3 -cpus 15 maker_Braula3_opts.ctl maker_bopts.ctl maker_exe.ctl
```
7.Train Augustus
* Follow SNAP steps till the export.ann and export.dna files.
> Note: AUGUSTUS uses the GeneBank GBK format file as input. 
```bash
gff3_merge -d Braula.round3_master_datastore_index.log -o Braula.round3.all.maker.gff
fasta_merge -d Braula.round3_master_datastore_index.log
maker2zff -n Braula.round3.all.maker.gff
fathom genome.ann genome.dna -validate > snap_validate_output.txt
cat snap_validate_output.txt | grep "error"
grep -vwE "MODEL_ID" genome.ann > genome.ann2
fathom genome.ann2 genome.dna -validate 
fathom genome.ann2 genome.dna -categorize 1000
fathom uni.ann uni.dna -export 1000 -plus           
```
* Use the Perl script zff2augustus_gbk.pl which converts SNAP ZFF files to GBK files
```bash
perl zff2augustus_gbk.pl > augustus.gbk
```
> Note: The script is available at https://github.com/hyphaltip/genome-scripts/blob/master/gene_prediction/zff2augustus_gbk.pl
* Split the augustus.gbk file into a training and a test set
```bash
randomSplit.pl augustus.gbk 100
```
> Note: Copy directory where there is the Augustus config path to be able to write in it
```bash
cp -R /config /train_augustus/round1/
```
* Create a new species for the Augustus training
```bash
new_species.pl --species=Braula_coeca --AUGUSTUS_CONFIG_PATH=/train_augustus/round1/config
```
* Train Augustus with the training set file, evaluate the training and save the output of this test to a file for later reference
```bash
etraining --species=Braula_coeca augustus.gbk.train --AUGUSTUS_CONFIG_PATH=/train_augustus/round1/config > training_trainingSet_rnd1
augustus --species=Braula_coeca augustus.gbk.test --AUGUSTUS_CONFIG_PATH=/train_augustus/round1/config | tee first_training.out > training_testSet_rnd1
```
* Improve prediction parameters of the models using the optimize_augustus.pl script
```bash
nohup nice -n15 optimize_augustus.pl --species=Braula_coeca augustus.gbk.train --AUGUSTUS_CONFIG_PATH=/train_augustus/round1/config --cpus=40 &
```
* Retrain and test Augustus with the optimized parameters and compare the results to the first run
```bash
etraining --species=Braula_coeca augustus.gbk.train --AUGUSTUS_CONFIG_PATH=/train_augustus/round1/config > training_trainingSet_rnd2
augustus --species=Braula_coeca augustus.gbk.test --AUGUSTUS_CONFIG_PATH=/train_augustus/round1/config | tee second_training.out > training_testSet_rnd2
```
8. Fourth round of gene prediction with Maker
* Options in the script maker_Braula4_opts.ctl to generate the maker_Braula4_opts.ctl file
```bash
#-----Genome (these are always required)
genome=Braula_assembly_21_09_12.fasta #genome sequence (fasta file or fasta embeded in GFF3 file)
organism_type=eukaryotic #eukaryotic or prokaryotic. Default is eukaryotic

#-----Re-annotation Using MAKER Derived GFF3
maker_gff= Braula.round3.all.maker.gff #MAKER derived GFF3 file
est_pass=1 #use ESTs in maker_gff: 1 = yes, 0 = no
altest_pass=0 #use alternate organism ESTs in maker_gff: 1 = yes, 0 = no
protein_pass=1 #use protein alignments in maker_gff: 1 = yes, 0 = no
rm_pass=1 #use repeats in maker_gff: 1 = yes, 0 = no

#-----Protein Homology Evidence (for best results provide a file for at least one)
protein=  #protein sequence file in fasta format (i.e. from mutiple oransisms)

#-----Repeat Masking (leave values blank to skip repeat masking)
model_org= #select a model organism for RepBase masking in RepeatMasker
rmlib= #provide an organism specific repeat library in fasta format for RepeatMasker
repeat_protein= #provide a fasta file of transposable element proteins for RepeatRunner
rm_gff= #pre-identified repeat elements from an external GFF3 file
prok_rm=0 #forces MAKER to repeatmask prokaryotes (no reason to change this), 1 = yes, 0 = no
softmask=1 #use soft-masking rather than hard-masking in BLAST (i.e. seg and dust filtering)

#-----Gene Prediction
snaphmm= #SNAP HMM file
augustus_species=Braula_coeca #Augustus gene prediction species model
est2genome=0 #infer gene predictions directly from ESTs, 1 = yes, 0 = no
protein2genome=0 #infer predictions from protein homology, 1 = yes, 0 = no

#-----MAKER Behavior Options
pred_stats=1 #report AED and QI statistics for all predictions as well as models
```
* Run Maker
```bash
AUGUSTUS_CONFIG_PATH=/config nohup maker -base Braula.round4 -cpus 15 maker_Braula4_opts.ctl maker_bopts.ctl maker_exe.ctl &> run_maker_round4 &
```
* Merge the output files of the round 4 of Maker
```bash
gff3_merge -d Braula.round4_master_datastore_index.log -o Braula.round4.all.maker.gff
fasta_merge -d Braula.round4_master_datastore_index.log
```
* Formate gene names by creating a table with the correspondance between new gene names and names created by Maker
```bash
maker_map_ids --prefix BRA1_ --justify 8 Braula.round4.all.maker.gff > Braula.round4.all.maker.id.map
```
* Copy gff to change names in the copy (do it with all the files to rename)
```bash
cp Braula.round4.all.maker.gff Braula.round4.all.maker.renamed.gff
```
* Replace names in gff files
```bash
map_gff_ids Braula.round4.all.maker.id.map Braula.round4.all.maker.renamed.gff
map_gff_ids final.genome.scf.rnd4.all.maker.id.map final.genome.scf.rnd4.all.maker.noseq.renamed.gff #annotation sans sequences
map_gff_ids final.genome.scf.rnd4.all.maker.id.map final.genome.scf.rnd4.all.maker.noseq.noevidence.renamed.gff #annotation sans sequences + à quelle étape gène trouvé (Augustus...)
```
* Replace names in fasta headers
```bash
map_fasta_ids Braula.round4.all.maker.id.map Braula.round4.all.maker.proteins.renamed.fasta
map_fasta_ids Braula.round4.all.maker.id.map Braula.round4.all.maker.transcripts.renamed.fasta
```
9. Genome annotation quality
* Using the AED distribution for each round of Maker
> The script can be downloaded at https://github.com/mscampbell/Genome_annotation/blob/master/AED_cdf_generator.pl
```bash
perl AED_cdf_generator.pl -b 0.025 Braula.round1.all.maker.gff > AED_rnd1
perl AED_cdf_generator.pl -b 0.025 Braula.round2.all.maker.gff > AED_rnd2
perl AED_cdf_generator.pl -b 0.025 Braula.round3.all.maker.gff > AED_rnd3
perl AED_cdf_generator.pl -b 0.025 Braula.round4.all.maker.gff > AED_rnd4
```
> Compare AED of all rounds of Maker with R
>
>> Note: Following https://darencard.net/blog/2017-05-16-maker-genome-annotation/: "AED ranges from 0 to 1 and quantifies the confidence in a gene model based on empirical evidence. Basically, the lower the AED, the better a gene model is likely to be. Ideally, 95% or more of the gene models will have an AED of 0.5 or better in the case of good assemblies."
```bash
> round1 <- read.table("~/Documents/Projets_parasites/Braula_coeca/Annotation/AED_rnd1", header=TRUE)
> plot(round1,  col="red", pch="*", ylab = "Proportion of the gene models")
> round2 <- read.table("~/Documents/Projets_parasites/Braula_coeca/Annotation/AED_rnd2", header=TRUE)
> points(round2$AED, round2[[2]], col="blue", pch="+")
> round3 <- read.table("~/Documents/Projets_parasites/Braula_coeca/Annotation/AED_rnd3", header=TRUE)
> points(round3$AED, round3[[2]], col="orange", pch="x")
> round4 <- read.table("~/Documents/Projets_parasites/Braula_coeca/Annotation/AED_rnd4", header=TRUE)
> points(round4$AED, round4[[2]], col="grey")
> legend(0.8, 0.4, legend=c("round1", "round2", "round3", "round4"), col = c("red", "blue", "orange", "grey"), pch = c("*","+", "x", "o"))
```
* Using the proportion of proteins with a Pfam domain
> Remove stop codon (*) because interproscan cannot deal with them
```bash
sed 's/\*//g' Braula.round4.all.maker.proteins.renamed.fasta > Braula.round4.all.maker.proteins.renamed.nostop.fasta
```
> Calculate the number of proteins in the file
```bash
grep ">" -c Braula.round4.all.maker.proteins.renamed.nostop.fasta
```
> Look for Pfam domain
```bash
/opt/interproscan-5.46-81.0/interproscan.sh -i Braula.round4.all.maker.proteins.renamed.nostop.fasta -dp -appl Pfam
cut -f1 Braula.round4.all.maker.proteins.renamed.nostop.fasta.tsv | sort | uniq | wc -l
```
* Using grep on gff file to count the number of annotated genes
```bash
grep $'\tmaker\t' Braula.round4.all.maker.renamed.gff | cut -f3 | sort | uniq
```
> Categories annotated by Maker are: CDS - exon - five_prime_UTR - gene - mRNA - three_prime_UTR
```bash
grep -c $'\tgene\t' Braula.round4.all.maker.renamed.gff
```
* Using BUSCOv5.0.0
> On the assembly
```bash
busco -i Braula_assembly_21_09_12.fasta -o Busco_Braula5.0.0 -m geno -c 36 -l diptera
```
> On the final annotation
```bash
busco -i Braula.round4.all.maker.proteins.renamed.fasta -o Busco_Braula5.0.0 -m prot -c 36 -l diptera
```
## Phylogenomic analysis of the Ephydroidea
1. Assembly of genomic reads of _Cacoxenus indagator_ and _Rhinoleucophenga_ cf. _bivisualis_
With MaSurCa:
```bash
Illumina paired end <fragment mean> <fragment stdev>
PE= pe 350 50 input1 input2

default parameters: 
EXTEND_JUMP_READS=0
GRAPH_KMER_SIZE = auto
USE_LINKING_MATES = 1
USE_GRID=0
GRID_ENGINE=SGE
GRID_QUEUE=all.q
GRID_BATCH_SIZE=500000000
LHE_COVERAGE=25
LIMIT_JUMP_COVERAGE = 300
CA_PARAMETERS =  cgwErrorRate=0.15
CLOSE_GAPS=1
NUM_THREADS = 32
SOAP_ASSEMBLY=0
FLYE_ASSEMBLY=0

modify parameter: 
#this is mandatory jellyfish hash size -- a safe value is estimated_genome_size*20 
#(Drosophila melanogaster #180Mb*20)
JF_SIZE = 3600000000

2. Assembly of RNA-Seq reads of _Acletoxenus_ sp.
The assembly was run in Galaxy Europe, (https://training.galaxyproject.org/training-material/topics/transcriptomics/tutorials/full-de-novo/tutorial.html#assembly-with-trinity) using default parameters 
Step 1: Input dataset
SRR10694693 - fastq = Select at Runtime.
Step 2: Trinity
Paired or Single-end data? = Paired-end
Strand specific data = False
Jaccard Clip options = False
Run in silico normalization of reads = False
Additional Options:
Minimum Contig Length = 200
Use the genome guided mode? = No
Error-corrected or circular consensus (CCS) pac bio reads = Select at Runtime.
Minimum count for K-mers to be assembled = 1

#Busco of all ephydroid assemblies
busco -i [ephydroid_assembly] -o [output] -m geno -l diptera

#Creation of a supermatrix out single-copy conserved busco sequences from all species
perl Supermatrix_busco.pl
#Phylogeny inference
iqtree -s [protein alignment] -bb 1000 -nt AUTO -m JTT+F+R6 -redo -safe
#The JTT+F+R6 model was inferred using IqTree from a previous analysis only using single-copy-orthologs inferred by OrthoFinder of dipteran sequences (see below).

#Time calibration of the ephydroid tree using MCMCTree
Use tree file:
20 1
((((((Acletoxenus_sp,Rhinoleucophenga_sp),(Braula_coeca,(((Cacoxenus_indagator,Phortica_variegata),Leucophenga_varia)'>0.121<0.47',Stegana_sp))'>0.378<0.538'),((Chymomyza_costata,Scaptodrosophila_lebanonensis),((Drosophila_busckii,(Drosophila_grimshawi,(Drosophila_mojavensis,Drosophila_virilis)'>0.098<0.279')'>0.157<0.364')'>0.296<0.441',((Drosophila_melanogaster,Drosophila_pseudoobscura)'>0.043<0.172',Drosophila_willistoni)'>0.154<0.407')'>0.361<0.483')'>0.501<0.56')'>0.552<0.821',Cryptochetum_sp),(Curtonotum_sp,Diastata_repleta)),Ephydra_gracilis)'>0.695<1.126';
//end of file
 
##Reconstruction of ecological niches using BayesTraits
BayesTraitsV4 Ephydroidea_calibrated.tre Ephydroidea_niche.txt < EphydroideaIN.txt

##Phylogenomic analysis of 17 dipteran and 25 hymenopteran genomes
#Extract primary transcripts for all genomes using OrthoFinder from transcripts faa files
for f in *faa ; do python3 OrthoFinder/tools/primary_transcript.py 
#Create orthogroups
orthofinder -f primary_transcripts/
#Creation of a supermatrix out of single-copy-orthologs sequences for each orthogroup from the output folder Single_Copy_Orthologue_Sequences
perl Supermatrix_OG.pl
#Phylogeny inference
iqtree -s [protein alignment] -bb 1000 -nt AUTO -m JTT+F+R6 -redo -safe
#The JTT+F+R6 model was inferred using IqTree from a previous analysis only using single-copy-orthologs of dipteran sequences.

#Time calibration of the tree using MCMCTree
Use tree file:
42	1
(((Diachasma_alloeum,Fopius_arisanus),((Leptopilina_boulardi,Nasonia_vitripennis),((Monomorium_pharaonis,(((Colletes_gigas,Hylaeus_anthracinus),(Dufourea_novaeangliae,(Megalopta_genalis,Nomia_melanderi))),((Megachile_rotundata,Osmia_bicornis),(Habropoda_laboriosa,(Ceratina_calcarata,(Eufriesea_mexicana,((((Bombus_affinis,Bombus_pyrosoma),Bombus_impatiens),Frieseomelitta_varia),(Apis_florea,((Apis_dorsata,Apis_laboriosa),(Apis_cerana,Apis_mellifera))))))))))'>0.9<1.2',Vespula_vulgaris))),((((Bacterocera_dorsalis,Ceratitis_capitata),Rhagleotis_pomonella),(((((Drosophila_busckii,(Drosophila_grimshawi,(Drosophila_mojavensis,Drosophila_virilis))),((Drosophila_melanogaster,Drosophila_pseudoobscura),Drosophila_willistoni)),Scaptodrosophila_lebanonensis)'>.5<.56',(Braula_coeca,(Leucophenga_varia,Phortica_variegata))),Ephydra_gracilis)),(Lucilia_cuprina,Stomoxys_calcitrans)));
//end of file

#Busco of all ephydroid assemblies
busco -i [dipteran_assembly] -o [output] -m geno -l diptera
busco -i [hymenopteran_assembly] -o [output] -m geno -l hymenoptera


##Reconstruction of ancestral states using Phylotools in R
> library(phytools)
> G <- read.table("Braula_GC.txt",row.names=1) #input TSV file with 2 columns first taxon name and then value (e.g., genome size, gene content, etc.)
> GS <- as.matrix(G)[,1]
> T <- read.nexus("tree1.tre") #time-calibrated tree file in nexus format
> fit <- fastAnc(T,GS,vars=TRUE,CI=TRUE)
> obj<-contMap(T,GS,plot=FALSE)
> plot(obj,legend=0.7*max(nodeHeights(T)),fsize=c(0.7,0.9))

##Horizontal Transposon Transfer analyses
#In order to detect possible HTT between Braula coeca and Apis mellifera, we used the B. coeca whole genome as query to perform a blastn similarity search against the whole A. mellifera genome (all default options, including “-task megablast”): blastn -query BraulaCoecaGenome.fas -db ApisMelliferaGenome.fas -outfmt 6 -out outputblastnBraulaGenomeVersusApisMelliferaGenome.txt

#the output file is then parsed in R

library(data.table)
library(Biostrings)
library(taxonomizr)

dt=fread("outputblastnBraulaGenomeVersusApisMelliferaGenome.txt")

#filter out hits shorter than 300 bp and/or with an evalue higher than 0.0001
dt2=dt[dt$V4>299]

setorder(dt2, V1, V7, V8)

#generate table to merge coordinates on braula scaffolds
dt2$strand="+"

dt2$num=c(1:nrow(dt2))

#create BED with name of Braula coeca contig, start and end coordinate of blast hits to merge them (in cases where two or more regions of A. mellifera align on overlapping regions of B. coeca)
toMergedt2=as.data.table(cbind(dt2$V1, dt2$V7, dt2$V8, dt2$num, dt2$V12, dt2$strand))
write.table(toMergedt2, "toMergedt2.txt", row.names = F, col.names = F, quote = F, sep = "\t")

#dos2unix toMerge2.txt
#bedtools merge -s -c 4 -o distinct -i toMergedt2.txt  > toMergedt2.txt.merged
#non redundant coordinates of blast hits were manually retrieved from toMergedt2.txt.merged in an excel spreadsheet
#B. coeca sequences involved in blast hits were retreived from the genome using seqtk subseq Braula_assembly_21_09_12.fasta coord.txt > seq.fas
#These sequences were clustered at 80% ID threshold using vsearch --cluster_fast seq.fas --consout xxx2  --id 0.8 --msaout msaout2 --iddef 0 --strand both
#The 50 resulting consensus sequences were loaded in R to rename them

seq=readDNAStringSet("50consensus.fas")

num=c(1:50)

names(seq)=num

writeXStringSet(seq, "seqRenamed.fas", format = "fasta")

#A diamond blastx similarity search was then performed using the 50 consensus sequences as queries on a database of transposable element proteins provided in the RepeatModeler pipeline (Flynn et al. 2020) in order to identify transposable elements  
#nohup diamond blastx  --min-score 40 --more-sensitive --outfmt 6 qseqid sseqid pident length mismatch gapopen qstart qend sstart send evalue bitscore stitle  -d /mnt/35To/db/NR_diamond/nr.dmnd -q seqRenamed.fas -o outputDiamondblastX_50consensus_Versus_nr &

#manually select all consensus sequences that blast against a TE (here, all blast against FAMAR and related TEs)

#The FAMAR TE was retrieved from repbase and used as a query to perform a similarity search using online blastn (megablast) against Rodentia, human, afrotheria, insectivora, chiroptera, carnivora, trichoptera, psocodea, hymenoptera, lepidoptera, diptera, neuroptera, dermaptera, arachnida, chelicerata, annelida, nematoda, platyhelminthes, myriapoda, cnidaria, planaria)

#all blastn outputs were downloaded from NCBI

#comment lines were removed and all outputs were concatenated in all.txt file, which was then loaded in R

blout=fread("famarBlastnOutputs/all.txt", fill = TRUE)

#remove empty lines
blout=blout[!is.na(blout$V3)]

#obtain taxonomy info for each blast hit using NCBI acccession number and taxonomizr package (make sure to load all taxonomizr databases)
ids<-accessionToTaxa(blast$V2,'accessionTaxa.sql')
taxo=getTaxonomy(ids,'accessionTaxa.sql')
blast=cbind(blast, taxo)
#write.table(blast, "blout_taxo.txt", col.names = F, row.names = F, quote = F, sep = "\t")
#blout_taxo=fread("blout_taxo.txt")

#only retain hits longer than 900 bp and >79.9% ID
blout_taxo2=blout_taxo[blout_taxo$V4>900  & blout_taxo$V3 > 79.9]

#copies are then recursively numbered for each species
setorder(blout_taxo2, V19, -V12)

blout_taxo2$numbering <- ave(blout_taxo2$V1,  
                       blout_taxo2$V19,
                       FUN = seq_along)

blout_taxo2$numbering=as.numeric(blout_taxo2$numbering)

blout_taxo2_noNA=blout_taxo2[!is.na(blout_taxo2$V19)]

#We then retain only the ten best hits per species
TenBestFamarHits=blout_taxo2_noNA[blout_taxo2_noNA$numbering > 0 & blout_taxo2_noNA$numbering < 11]

TenBestFamarHits$num=c(1:nrow(TenBestFamarHits))

#we make a table with accession numbers of contigs containing the then best hits per species and retrieve those contigs from Genbank
accessionTenBestFamarHits=as.data.table(TenBestFamarHits$V2)

write.table(accessionTenBestFamarHits, "accessionTenBestFamarHits.txt", col.names = F, row.names = F, quote = F, sep = "\t")

#we make a table with coordinate of ten best hits
coordTenbestFamarHits=as.data.table(cbind(TenBestFamarHits$V2, ifelse(TenBestFamarHits$V9<TenBestFamarHits$V10, TenBestFamarHits$V9, TenBestFamarHits$V10), ifelse(TenBestFamarHits$V9<TenBestFamarHits$V10, TenBestFamarHits$V10, TenBestFamarHits$V9)) )

write.table(coordTenbestFamarHits, "coordTenbestFamarHits.txt", col.names = F, row.names = F, quote = F, sep = "\t")

#we retrieve the sequences of the ten best hits in all species using seqtk subseq ContigTenBestHits.fasta coordTenbestFamarHits.txt > TenbestFamarHitsOtherSpecies.fas

#formatting to make a correspondence between names of the retrieved sequences (TenbestFamarHitsOtherSpecies.fas) and the TenBestFamarHits table
TenBestFamarHits$namCor=paste(TenBestFamarHits$V2, ifelse(TenBestFamarHits$V9<TenBestFamarHits$V10, TenBestFamarHits$V9+1, TenBestFamarHits$V10+1), sep = ":")
TenBestFamarHits$namCor2=paste(TenBestFamarHits$namCor, ifelse(TenBestFamarHits$V9<TenBestFamarHits$V10, TenBestFamarHits$V10, TenBestFamarHits$V9), sep = "-")

seq=readDNAStringSet("TenbestFamarHitsOtherSpecies.fas")

n=as.data.table(names(seq))

n2=as.data.table(tstrsplit(n$V1, " ", fixed = TRUE))

names(seq)=n2$V1

seq2=as.data.table(seq)

seq2$names=n2$V1

#Add sequences to the TenBestFamarHits table and reverse those that need to be reverse in order to align them all
TenBestFamarHits$seq=seq2$x[match(TenBestFamarHits$namCor2, seq2$names)]

TenBestFamarHits$toreverse=TenBestFamarHits$V9>TenBestFamarHits$V10

toRev=TenBestFamarHits[TenBestFamarHits$toreverse==T]

toRev_seq=DNAStringSet(toRev$seq)
toRev_rev=reverseComplement(toRev_seq)
names(toRev_rev)=paste(gsub(" ", "_", toRev$V19), toRev$num, sep = "_")

fwd=TenBestFamarHits[TenBestFamarHits$toreverse==F]
fwd_seq=DNAStringSet(fwd$seq)

names(fwd_seq)=paste(gsub(" ", "_", fwd$V19), fwd$num, sep = "_")

#generate a file with all ten best hits for all species in order to align them, trim them and submit them to phylogenetic analysis
all_seq = DNAStringSet(c(fwd_seq, toRev_rev))

writeXStringSet(all_seq, "allFamarSeqOtherSpecies2.fas", format = "fasta")

#alignement was done with muscle: nohup muscle -in allFamarSeqOtherSpecies2Braula.fas -out allFamarSeqOtherSpecies2Braula.fas.aligned

#phylogeny inference
iqtree -s allFamarSeqOtherSpecies2Braula.fas.aligned -bb 1000 -nt AUTO -redo -safe

##Gene family evolution using Cafe5 using orthogroups or genegroups (to convert orthogroups into genegroups cf. OG2GG.pl)
cafe5 -t Braula_tree.txt -i genegroups_genecount.tab -e
cafe5 -t Braula_tree.txt -i genegroups_genecount2.tab -p -l 0.41 -eerror_model_0.05.txt

##Gustatory and odorant receptors annotations using InsectOR
exonerate --model protein2genome --maxintron 300 [protein query sequence fasta file] [genome fasta file] -p pam250 --showtargetgff TRUE
#The exonerate site was then analyzed on the InsectOR site, with option 7tm_6 activated. Outputs renamed GR.fas and OR.fas for gustatory and odorant receptors amino acid sequences, respectively.
mafft GR.fas >GR_aln.fas
mafft OR.fas >OR_aln.fas
iqtree -s GR_aln.fas -bb 1000 -redo
iqtree -s OR_aln.fas -bb 1000 -redo
