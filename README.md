# Building trees from low-input DNA

## Pipeline steps

This [Nextflow](https://www.nextflow.io/) pipeline was written to generate
phylogenetic trees from low input WGS/WES. It takes as input `CaVEMAN` (SNV) 
and `Pindel` (indel) VCF files generated from the alignment files from 
sequencing. The steps are as follows.

1. Reflag the VCFs to remove some filters that have been found to be too 
stringent. The flags that are removed depend on the type of experiment (WGS vs
WES vs TGS) and the variant type (SNV vs indel).

2. Filter the variants.

  a. Filter the `CaVEMAN` SNV output. We run Mathis Sanders' 
  [`SangerLCMFiltering`](https://github.com/MathijsSanders/SangerLCMFiltering) 
  a.k.a. [`hairpin`](https://confluence.sanger.ac.uk/display/CAS/hairpin). This 
  filters out SNVs based on ASMD and CLPM values to remove cruciform DNA 
  artifacts that commonly form in low input DNA sequencing (FILTER = PASS & 
  CLPM=0.00 & ASRD>=0.87). 

  b. Filter the `Pindel` indel output (FILTER = PASS).

3. Run [`cgpVAF`](https://confluence.sanger.ac.uk/pages/viewpage.action?pageId=22710418)
a.k.a. [`vafCorrect`](https://github.com/cancerit/vafCorrect). This revisits
variants in individual samples that were non-significant, but that were found to
be significant in other samples of the same donor. `cgpVAF` also facilitates
unbiased `pileup` (SNVs) and `exonerate` (indels)-based VAF calculation for the
union of variant sites in the set of related samples from each donor. 

4. Run [Sequoia](https://github.com/TimCoorens/Sequoia) for tree building.

## Requirements

In order to run the pipeline you must have 
[Nextflow](https://www.nextflow.io/docs/latest/install.html) and 
[Singularity](https://docs.sylabs.io/guides/3.5/user-guide/introduction.html) 
installed. If running from the Sanger farm, simply load the Singularity module 
before running the Nextflow command:

```
$ module load singularity
```

## Usage

Here is an example command to run the pipeline:

```
$ nextflow run . -profile test -w test/work/
```

For a list of all parameters, run the command with the `--help` option:

```
$ nextflow run . --help

 N E X T F L O W   ~  version 24.04.2

Launching `./main.nf` [maniac_venter] DSL2 - revision: 51676fd9a6

Typical pipeline command:

  nextflow run low_input_trees --sample_sheet sample_sheet.csv --sequencing_type WGS --outdir out/

Input/output options
  --sample_sheet                           [string]  Path to comma-separated file containing metadata and file paths the samples in the experiment.
  --sequencing_type                        [string]  The type of sequencing done in the project. Must be one of "WGS" (whole genome sequencing) or "WES" (whole 
                                                     exome sequencing). Default is WGS. (accepted: WGS, WES) [default: WGS] 
  --outdir                                 [string]  The output directory where the results will be saved.
  --email                                  [string]  Email address for completion summary.

Reference genome options
  --genome_build                           [string]  The genome build. Must be one of "hg19" or "hg38". (accepted: hg19, hg38) [default: hg38]
  --fasta                                  [string]  Path to FASTA genome file. A matching index file (*.fai) must be present in the same directory. 
                                                     [default: /lustre/scratch126/casm/team273jn/share/pileups/reference_data/hg38/genome.fa] 

hairpin options
  --snp_database                           [string]  null [default: /lustre/scratch126/casm/team273jn/share/pileups/reference_data/hg38/SNP.vcf.gz]
  --fragment_threshold                     [integer] null [default: 4]

cgpVAF options
  --cgpVAF_normal_bam                      [string]  Path to the in silico normal BAM. [default: 
                                                     /nfs/cancer_ref01/nst_links/live/2480/PDv38is_wgs/PDv38is_wgs.sample.dupmarked.bam] 
  --high_depth_bed                         [string]  Path to a reference BED file of high depth regions. [default: 
                                                     /lustre/scratch126/casm/team273jn/share/pileups/reference_data/hg38/highdepth.bed.gz] 

Sequoia options
  --sequoia_beta_binom_shared              [boolean] Only run beta-binomial filter on shared mutations. If FALSE, run on all mutations, before germline / depth 
                                                     filtering [default: true] 
  --sequoia_normal_flt                     [string]  Name of the dummy normal to exclude from cgpVAF output
  --sequoia_snv_rho                        [number]  Rho value threshold for SNVs [default: 0.1]
  --sequoia_indel_rho                      [number]  Rho value threshold for indels [default: 0.15]
  --sequoia_min_cov                        [integer] Lower threshold for mean coverage across variant site [default: 10]
  --sequoia_max_cov                        [integer] Upper threshold for mean coverage across variant site [default: 500]
  --sequoia_only_snvs                      [boolean] If indel file is provided, only use SNVs to construct the tree (indels will still be mapped to 
                                                     branches) [default: true] 
  --sequoia_keep_ancestral                 [boolean] Keep an ancestral branch in the phylogeny for mutation mapping
  --sequoia_exclude_samples                [string]  Option to manually exclude certain samples from the analysis, separate with a comma
  --sequoia_cnv_samples                    [string]  Samples with CNVs, exclude from germline / depth-based filtering, separate with a comma
  --sequoia_vaf_absent                     [number]  VAF threshold (autosomal) below which a variant is absent [default: 0.1]
  --sequoia_vaf_present                    [number]  VAF threshold (autosomal) above which a variant is present [default: 0.3]
  --sequoia_mixmodel                       [boolean] Use a binomial mixture model to filter out non-clonal samples?
  --sequoia_min_clonal_mut                 [integer] If using binomial mixture model, minimum number of clonal mutations (in cluster higher than 
                                                     --VAF_treshold_mixmodel) needed to include sample [default: 35] 
  --sequoia_tree_mut_pval                  [number]  Pval threshold for treemut's mutation assignment [default: 0.01]
  --sequoia_genotype_conv_prob             [boolean] Use a binomial mixture model to filter out non-clonal samples?
  --sequoia_min_pval_for_true_somatic      [number]  Pval threshold for somatic presence if generating a probabilistic genotype matrix [default: 0.05]
  --sequoia_min_variant_reads_shared       [integer] Minimum variant reads used in generating a probabilistic genotype matrix [default: 2]
  --sequoia_min_vaf_shared                 [integer] Minimum VAF used in generating a probabilistic genotype matrix [default: 2]
  --sequoia_create_multi_tree              [boolean] Convert dichotomous tree from MPBoot to polytomous tree [default: true]
  --sequoia_mpboot_path                    [string]  Path to MPBoot executable [default: ./]
  --sequoia_germline_cutoff                [integer] Log10 of germline qval cutoff [default: -5]
  --sequoia_plot_spectra                   [boolean] Plot mutational spectra?
  --sequoia_max_muts_plot                  [integer] Maximum number of SNVs to plot in mutational spectra [default: 5000]
  --sequoia_lowVAF_filter                  [integer] Minimum VAF threshold to filter out subclonal variants. Disabled by default. [default: 0]
  --sequoia_lowVAF_filter_positive_samples [integer] Read number to apply exact binomial filter for samples with more than given number of reads. Disabled by 
                                                     default. [default: 0] 
  --sequoia_VAF_treshold_mixmodel          [number]  VAF threshold for the mixture modelling step to consider a sample clonal [default: 0.3]

 !! Hiding 22 params, use the 'validation.showHiddenParams' config value to show them !!
------------------------------------------------------
```