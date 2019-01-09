# FunGAP: Fungal Genome Annotation Pipeline v1.1.0

**Last updated: Jan 7, 2019**

Publication: https://academic.oup.com/bioinformatics/article/33/18/2936/3861332

FunGAP performs gene prediction on given genome assembly and RNA-seq reads. See **INSTALL.md** and **USAGE.md** for installation and usage instruction, or you can go to wiki tab for the same.

* [FunGAP INPUT & OUTPUT](#inputoutput)
* [Pipeline description](#pipedesc)
  * [Step1: Preprocessing of input data](#step1)
  * [Step2: Gene prediction](#step2)
  * [Step3: Gene model evaluation and filtration](#step3)
* [Contact](#contact)

<a name="inputoutput"></a>
## FunGAP INPUT & OUTPUT

FunGAP inputs:
```
--output_dir                      | Output directory
--trans_read_files                | Illumina paired-end mRNA reads files (FASTQ)
--genome_assembly                 | Genome assembly file (FASTA)
--augustus_species                | Augustus --species argument
--sister_proteome                 | Protein database (FASTA)
--num_cores                       | Number of CPU cores to be used
```
FunGAP outputs:
```
fungap_output.gff3                | Tab-delimited genomic feature format
fungap_output_prot.faa            | Translated protein sequences
fungap_output_result_summary.html | Summary of FunGAP results
```

<a name="pipedesc"></a>
## Pipeline description

![](http://compbio.korea.ac.kr/bnmin/fungap/scheme_fungap_ver2.png)

<a name="step1"></a>
### Step1: Preprocessing of input data
In preprocessing step, FunGAP masks repeat regions in genome assembly (input data 1) and assembles mRNA reads into transcript contigs (input data 2). 

**Repeat masking**

Repeat masking is a crucial step in eukaryotic gene prediction because genomic regions, such as transposon repeats, often make false alignments and interfere with gene prediction. FunGAP employs a repeat masking procedure embedded in the Maker pipeline along with a genome-specific repeat library built by RepeatModeler (http://www.repeatmasker.org/RepeatModeler.html).

**Assembly of mRNA reads**

User-provided mRNA reads are assembled by the Trinity program. A BAM-format file for genome-guided assembly is generated by a Hisat2 read aligner and Samtools format converter (SAM file to sorted BAM file). An optional parameter ```--jaccard_clip``` in Trinity is used for fungal transcript assembly because high gene density leads to UTR overlap in the assembly. This option helps avoid fusion of neighbor transcripts. The maximum intron length is set to 2000 bp with the ```--max-intronlen``` option in Hisat2.

<a name="step2"></a>
### Step2: Gene prediction

FunGAP uses three gene prediction tools: Augustus, Braker, and Maker. The outcomes of predictions are stored in GFF3 and FASTA files for the next set of evidence score calculations.

**Maker and default parameters used by FunGAP**

FunGAP runs Maker four times with iterative SNAP gene model training, as previously described. FunGAP uses the correct_est_fusion option to correct fusion of neighbor transcripts in mRNA assembly due to the above-mentioned high gene density of fungal genomes. Maximum intron length is set to 5000 bp with the ```split_hit``` option. Single-exon genes longer than 50 amino acids are predicted by setting the single_exon and single_length options.

**Augustus and default parameters used by FunGAP**

FunGAP runs Augustus with the augustus_species parameter specified by a user. The option ```--softmasking``` is turned on as repeat-masking generates soft-masked assembly. To allow overlapping CDS predictions, FunGAP turns on the ```--singlestrand``` option. The output is GFF3, and translated protein sequences are generated in FASTA by a simple parsing script.

**Braker and default parameters used by FunGAP**

Braker performs unsupervised RNA sequencing-based genome annotation using GeneMark-ET and Augustus. The option ```--softmasking``` is turned on as repeat-masking generates soft-masked assembly. The input file for Braker is the mRNA reads alignment formatted in a BAM file produced in the preprocessing step.

<a name="step3"></a>
### Step3: Gene model evaluation and filtration

In the previous step, three gene predictors generated a set of predicted genes (designate as “gene models” hereafter). FunGAP produces “non-overlapping” coding sequences by evaluating all gene models and retaining only best-scored models. The evaluation is performed by three tools: BLASTp, Benchmarking Universal Single-Copy Orthologs (BUSCO), and InterProScan. Bit scores from alignments are multiplied by length coverage because longer gene models have more chances to get higher alignment scores. The sum of three scaled bit scores becomes the evidence score for each gene model. Finally, the filtration produces a final set of gene models.

**BLASTp**

Sequence similarity with genes in phylogenetically close genomes can be an evidence for predicted genes being actual genes. Users provide the proteome of phylogenetically related organisms with the ```--sister_proteome``` argument. For convenience, FunGAP provides a script, download_sister_orgs.py, which downloads protein sequences from NCBI for a given taxon. To reduce computing time, FunGAP integrates the gene models from three gene predictions, and removes identical gene models to make nonredundant gene models.

**BUSCO**

BUSCO provides hidden Markov models for single-copy orthologs conserved in all fungal genomes. Evidence scores for BUSCO are calculated by multiplying “full sequence scores” in hmmer output and length coverage [min (query length, target length)/max (query length, target length)].

**InterProScan (Pfam domain prediction)**

Pfam provides a database of manually curated protein families. We assume that gene models annotated with a Pfam domain are more likely to be an actual gene. Evidence scores for Pfam are directly provided by the hmmer3-match score in the XML output of InterProScan (-f XML option). For multiple domains in one gene model, the sum of the scores is used.

**BLASTn**

Sequence similarity with assembled transcriptome can give the direct evidence for reliability of predicted genes. FunGAP runs BLASTn for each predicted gene against Trinity-assembled transcripts. Length coverage is also considered.

****

**Scoring function**

Three bit scores gained from the above four sources are summed to provide evidence scores for each gene model. The equation of this scoring function is as follows:

Evidence score (gene model) = BLASTp_score*cov(query)*cov(target) + BUSCO_score + Pfam_scores + BLASTn_score*cov(query)*cov(target)

**Filtration**

In the filtration process, FunGAP finds “gene blocks” defined as a set of gene models that overlap with at least one base pair. FunGAP gets all combinations of gene models in a gene block and calculates the sum of the evidence scores. Gene models in the block with the highest evidence score are selected as final genes of that region. Short coding sequence overlap (less than 10% of coding sequence length) is allowed.

![](http://compbio.korea.ac.kr/bnmin/fungap/filtering.png)

<a name="contact"></a>
## Contact

* Project principal investigator: Prof. In-Geol Choi, CSBL at Korea University
* Contact (email-address): igchoi at korea.ac.kr or mbnmbn00 at korea.ac.kr (or mbnmbn00 at gmail.com)

If you have any problem to install or run, please don't hesistate to contact us. We will help you as much as we can. Any input from users will help to build more robust pipeline.
