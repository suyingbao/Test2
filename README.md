# Identification of disease-critical tissues and cell types 

## Table of Contents
- [Description](#description)
- [Overview](#overview)
- [Getting started](#getting-started)
- [Output](#output)
- [Data resource](#data-resource)
    - [Profiles for tissue enrichment](#profiles-for-tissue-enrichment)
    - [scRNA-seq for cell type enrichment](#scrna-seq-for-cell-type-enrichment)
- [Caveats](#caveats)
- [Contact](#contact)

## Description <a name="description"></a>
This page outlines our comprehensive tissue and cell type enrichment analysis pipeline, designed to pinpoint disease-relevant tissues and cell types. This is achieved by integrating data from bulk RNA-seq, single-cell RNA-seq (scRNA-seq), epigenomic chromatin profiles, epigenomic SNP-to-gene maps, and genome-wide association study (GWAS) summary statistics. 

For tissue enrichment analysis, we employ stratified linkage disequilibrium score **(sLDSC)** regression to assess whether disease heritability is enriched in regions surrounding genes with the highest specific expression or those exhibiting the strongest chromatin activity within a given tissue. See [Overview](#overview) step 1. 


In cases where the given GWAS was significantly enriched in a tissue across at least three different profiles (defined as **Enriched tissue**, See [Overview](#overview) step 2), or when the tissue enrichment is observed with Coefficient_P_value<0.01 in two profiles, or achieves Bonferroni significance correction in one assay (defined as **Possibly enriched tissue** ), we conduct cell type enrichment analysis using scRNA-seq data from the corresponding donor tissue. This analysis is performed using our in-house developed **sclinker2** model, which significantly enhances the analytical power and reduces the false discovery rate compared to the previously released [sclink model](https://www.nature.com/articles/s41588-022-01187-9). See [Overview](#overview) step 3. 

For cell type(s) with significant enrichment (defined as **Enriched celltype**, with Coefficient_P_value<0.01), we further use [MAGMA](https://cncr.nl/research/magma/) to prioritize genes which drive the heritability enrichment within the cell type identified as enriched in the previous step. The top 3 genes will be selected as the **Top_genes**. See [Overview](#overview) step 4. 

All analysis results are finally consolidated into a comprehensive table for easy reference and visualized in a Rshiny app.

## Overview <a name="overview"></a>

<!-- <img src="images/pipeline.png" alt="drawing" width="600">Overview of the pipeline</a> -->
<div style="text-align: center;">
     <h3>Overview of the Pipeline</h3>
    <img src="images/pipeline.png" alt="drawing" width="800">
</div>

## Getting started <a name="getting-started"></a>
With a specified YOGA ID, you can execute the entire pipeline on miniwdl by running:

```bash

# The current pipeline version is v2
bash v2/run_workflow.sh $YOGAID

```

## Output <a name="output"></a>

All the output for a given YOGA ID would be under: 
``` bash

cd /mnt/efs_v2/gide_compbio/users/suying.bao/tissueEnrich/output/${YOGAID}/

```
#### Example outputs for three YOGA traits

| Trait      | Tissue     | Tissue_Escore | Tissue_pval | Cell       | Cell_Escore | Cell_pval | Top_genes           |
|------------|------------|---------------|-------------|------------|-------------|-----------|---------------------|
| Trait1     | Tissue1    | 1.85(GTEx),2.4(H3K27ac),3.2(H3Kme3)          | 1e-3(GTEx),3.1e-5(H3K27ac),1.2e-5(H3Kme3)        | Cell1      | 0.90        | 0.002      | GeneA,GeneB,GeneC |
| Trait2     | Tissue2    | 0.75(H3K27ac)          | 0.05(H3K27ac)        | -      | -        | -      | - |
| Trait3     | Tissue3    | 1.65(GTEx),3.7(H3K36me3)          | 0.004(GTEx),1.1e-5(H3K36me3)        | Cell3      | 0.70        | 0.01      | NA |

#### Three alternative outputs  
* If no enriched tissue is identified, the pipeline will terminate at this point, and output: **"No enriched tissue for ${PHENO}"** in the output file.
* If the given GWAS has chi^2<1, then the pipeline will output: **"WARNING: mean chi^2<1. There might be not enough signal in the GWAS data to estimate the heritability"**
* If the given GWAS has chi^2>=1, but remains very close to 1, then there will a warning **"WARNING: mean chi^2 may be too small. There might be not enough signal in the GWAS data to estimate the heritability"** in the end of the file.

#### Three potential outcomes for tissue enrichment analysis:
* Enriched tissue: >=3 profiles with pval<0.01
* Possibly enriched tissue: 2 profiles with pval<0.01, or 1 assay passing Bonferroni significance correction
* -: No enriched tissue for all the remaining

#### Three potential outcomes for cell type enrichment:
* Enriched celltype: cell type with pval<0.01
* -: No enriched cell type
* NA: no corresponding tissue’s scRNA-seq data available

#### Additional notes
* If there is no significantly enriched cell type or no significant result in MAGMA gene level analysis, then the 'Top_genes' will be '-'
* Out of the 37 main tissues analyzed in this pipeline, only 7 have corresponding donor tissue's scRNA-seq data. Consequently, there are no cell type enrichment results or top gene results for traits that were enriched in the remaining 30 tissues

## Data resource <a name="data-resource"></a>
#### Profiles for tissue enrichment <a name="profiles-for-tissue-enrichment"></a>
We carried out a series of hamonization work to integrate tissues across different profiles, details can be found in <u>scripts/script_harmonize_profiles_tissue_subtissues.r</u>. To eliminate ambiguity arising from tissue naming and the various sub-regions of tissues, we manually grouped subtissues from the same donor tissue under a unified tissue name. To support tissue enrichment analysis across multiple profiles, we retained only the main tissues supported by >=4 profiles, resulting in 37 main tissues used in the current pipeline.  

The tissue expression and chromatin profiles used in this pipeline were summarized as follows: 

| Source             | Dataset    | Organism | Total Main Tissues | Total Tissues | Selected Main Tissues | Selected Tissues |
|--------------------|------------|----------|--------------------|---------------|-----------------------|------------------|
| [GTEx](https://www.gencodegenes.org/human/stats_26.html)               | GTEx | human    | 34                 | 53            | 23                    | 40               |
| [Roadmap](http://www.roadmapepigenomics.org/) & [ENTEx](https://www.encodeproject.org/entex-matrix/?type=Experiment&status=released&internal_tags=ENTEx)   | H3K27ac    | human    | 37                 | 92            | 35                    | 60               |
| [Roadmap](http://www.roadmapepigenomics.org/) & [ENTEx](https://www.encodeproject.org/entex-matrix/?type=Experiment&status=released&internal_tags=ENTEx)   | H3K36me3   | human    | 39                 | 111           | 37                    | 63               |
| [Roadmap](http://www.roadmapepigenomics.org/) & [ENTEx](https://www.encodeproject.org/entex-matrix/?type=Experiment&status=released&internal_tags=ENTEx)   | H3K4me1    | human    | 36                 | 108           | 34                    | 61               |
| [Roadmap](http://www.roadmapepigenomics.org/) & [ENTEx](https://www.encodeproject.org/entex-matrix/?type=Experiment&status=released&internal_tags=ENTEx)   | H3K4me3    | human    | 41                 | 114           | 37                    | 64               |
| [Roadmap](http://www.roadmapepigenomics.org/) & [ENTEx](https://www.encodeproject.org/entex-matrix/?type=Experiment&status=released&internal_tags=ENTEx)   | H3K9ac     | human    | 17                 | 30            | 12                    | 18               |

Detailed tissue and corresponding main tissue (i.e. main tissue name for sub-tissues) after harmonization for each profile can be found <u>data/Tissue_profiles_info.csv</u>

#### scRNA-seq for cell type enrichment <a name="scrna-seq-for-cell-type-enrichment"></a>

The scRNA-seq datasets used in current pipeline were summarized as follows: 

| Dataset         | Organism | Donor Tissue | # of Cells | # of Individuals | # of Cell Types |
|-----------------|----------|--------------|------------|------------------|-----------------|
| [Adipose](https://www.biorxiv.org/content/10.1101/2020.04.19.049254v1)         | human    | adipose      | 11,184     | 3                | 13              |
| [PBMC (Zheng et al)](https://pubmed.ncbi.nlm.nih.gov/28091601/) | human    | blood        | 68,551     | 8                | 6               |
| [Brain](https://portal.brain-map.org/atlases-and-data/rnaseq/human-multiple-cortical-areas-smart-seq)           | human | brain | 47,509 | 3 | 9 |
| [Heart](https://pubmed.ncbi.nlm.nih.gov/32403949/)           | human    | heart        | 287,269    | 7                | 12              |
| [Kidney](https://www.kidneycellatlas.org/)          | human    | kidney       | 40,268     | 13               | 24              |
| [Liver](https://www.biorxiv.org/content/10.1101/2020.04.19.049254v1)           | human    | liver        | 13,340     | 4                | 12              |
| [Lung](https://advances.sciencemag.org/content/6/28/eaba1972.full)            | human    | lung         | 31,644     | 10               | 24              |


## Caveats <a name="caveats"></a>
* The LDSC model serves as the foundational model for conducting both tissue enrichment and cell type enrichment analyses. Since LDSC requires LD scores that are specific to the population being studied, using LD scores from a different population can lead to inaccurate results. Currently, our pipeline only support GWAS from EUR data. 
* The accuracy of LDSC can be affected by GWAS sample size and SNP heritability (𝑁×ℎ𝑔^2), and tissue/cell type specificity annotations
* LDSC assumes a polygenic model and that all SNPs contribute equally to the trait. This assumption might not be valid for traits influenced by a few large-effect loci
* Tissues or cell types exhibiting chromatin or gene expression profiles similar to those of a causal tissue or cell type will be identified as relevant to the disease. However, our multi-assay validation process significantly reduces the likelihood of erroneous associations.
* This analysis was limited by tissue/cell type gene expression/chromatin data available, and cannot rule out the importance of cell types not tested

## Contact <a name="contact"></a>
Please note that this repository will be continuosly updated. Please contact Suying Bao (suying.bao@regeneron.com) if you have any questions or would like to contribute to this repository.