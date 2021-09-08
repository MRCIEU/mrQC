
<!-- README.md is generated from README.Rmd. Please edit that file -->
# CheckSumStats

<!-- badges: start -->
<!-- badges: end -->
CheckSumStats is an R package for checking meta and summary data from genetic association studies. For example, the package can be used to check that the reported effect allele and effect allele frequency columns are correct. The package was developed for the Fatty Acids in Cancer Mendelian Randomization Collaboration (FAMRC). See our pre-print describing application of the package here: [Design and quality control of large-scale two-sample Mendelian randomisation studies](https://www.medrxiv.org/content/10.1101/2021.07.30.21260578v1)

## Installation

To install the latest version of CheckSumStats, perform as normal:

``` r
install.packages("devtools")
devtools::install_github("MRCIEU/CheckSumStats")
```

## General overview

This package exploits three groups of single nucleotide polymorphisms (SNPs) in order to identify potential errors or issues:

1.  **a 1000 genomes reference set**. This is a set of 2297 SNPs that have the same minor allele across the 1000 genomes super populations and that have a minor allele frequency between 0.1 and 0.3;
2.  **GWAS catalog associations**. These are SNPs that are associated with the trait of interest in the GWAS catalog; and
3.  the **test GWAS top hits**. These are SNPs that are strongly associated with the trait of interest in the target GWAS of interest.

Our objective is to extract summary data for these three groups of SNPs from the target GWAS of interest (we call this the test dataset) in order to perform the following quality control checks:

1.  confirm the identity of the effect allele frequency column
2.  confirm the identity of the effect allele column
3.  identify possible errors in the reported effect sizes
4.  infer ancestry

# Example 1. Glioma GWAS

In this example we show how the package can be used to check the accuracy of reported summary and metadata from a case-control genome-wide association study of glioma. In this particular example, the non-effect allele column was mis-labelled as the effect allele.

## Step 1. Extract and format the outcome summary data for glioma

The first step is to map the reported trait with the Experimental Factor Ontology (EFO).

``` r
library(CheckSumStats)
EFO<-get_efo(trait="glioma")
EFO
#> $efo_id
#> [1] "EFO_0000326" "EFO_0005543"
#> 
#> $confidence
#> [1] "confidence:GOOD"
```

Then we make a list of SNP rsids.

``` r
snplist<-make_snplist(efo_id=EFO$efo_id,trait="glioma",ref1000G_superpops=TRUE)
```

We now have a list of SNPs that include known genetic associations for glioma in the GWAS catalog as well as a 1000 genomes reference set. The number of reported genetic associations for glioma in the GWAS catalog is:

``` r
length(snplist)-2297 
# [1] 75
```

Next, we extract the summary associations statistics for these SNPs from the glioma dataset using the extract\_snps() function.

``` r
File<-system.file("extdata", "glioma_test_dat.txt", package = "CheckSumStats")
gli<-extract_snps(snplist=snplist,path_to_target_file=File,path_to_target_file_sep="\t")
```

In the above example, we extracted the summary data for the SNPs of interest from a tab separated text file that was stored on our local machine. In practice the File object is just the file path to your GWAS results. Note that the extract\_snps() function only works on MAC/linux machines. Alternatively, you could use the read.table() function to load all the summary data into R and then extract the SNPs of interest. Another alternative is to source the summary data from online databases, such as the Open GWAS project (<https://gwas.mrcieu.ac.uk/>). For example, the equivalent scripts to extract summary data for thyroid cancer from Open GWAS are:

``` r
EFO<-get_efo(trait="thyroid cancer")
snplist<-make_snplist(efo_id = EFO$efo_id,trait="thyroid carcinoma")
ao<-ieugwasr::gwasinfo()
Pos<-grep("thyroid cancer",ao$trait,ignore.case=TRUE)
ao$id[Pos]
#> [1] "ieu-a-1082"
thy <- ieugwasr::associations(id="ieu-a-1082", variants=snplist,proxies=0)  
```

Returning to the glioma example, having extracted the summary data for some SNPs, we now need to format the summary data. This is to get the data into the expected format for the QC tests to come.

``` r
Dat<-format_data(dat=gli,outcome="Glioma",population="European",pmid=22886559,study="GliomaScan",ncase="cases",ncontrol="controls",rsid="Locus",effect_allele="Allele1",other_allele="Allele2",or="OR",or_lci="OR_95._CI_l",or_uci="OR_95._CI_u",eaf="eaf.controls",p="p",efo="glioma")
```

In this example, the glioma results file contained columns for the odds ratio and 95% confidence intervals. In practice, the format of the effect size columns is highly variable across studies. For example, some studies may report the log odds ratio and its standard error or the odds ratio and P value without confidence intervals or a standard error. The format\_data() function accepts these and other effect size reporting formats. See ?format\_data() for more info.

Now we are ready to perform some quality checks on the data.

## <a id="step2"></a>Step 2. Check allele frequency metadata for the glioma GWAS

Next we create some plots to visualise potential problems with the effect allele frequency column. We do this by comparing allele frequency in the glioma dataset to the 1000 genomes super populations. The function harmonises the test and reference dataset to reflect the minor allele in the 1000 genomes superpopulations. Therefore, the presence of SNPs with allele frequency \> 0.5 in the test dataset implies an allele frequency conflict.

``` r
Plot1<-make_plot_maf(ref_1000G=c("AFR","AMR","EAS","EUR","SAS","ALL"),target_dat=Dat)
Plot1
```

!["README-example1\_mafplot.png"](/man/figures/README-example1_mafplot.png)

Data points with a red colour are SNPs with allele frequency conflicts. Allele frequencies in the glioma dataset are all greater than 0.5, indicating that the reported effect allele frequency column actually corresponds to the non-effect allele. See the "fasting glucose" GWAS example ([Other examples](#fasting_glucose)) for another type of allele frequency metadata error.

Notice also how conflicts are flagged across all SNPs across all superpopulations. This illustrates that allele frequency metadata errors can be identified without matching of test and reference datasets on ancestry. Notice also how the comparison provides information on the ancestral background of the test dataset: the test dataset is strongly correlated with the European-ancestry 1000 genomes super population, which matches the reported ancestry for the test dataset.

## Step 3. Check the effect allele metadata

We next check that the reported effect allele metadata is correct, by comparing the reported effect alleles for glioma to the GWAS catalog.

``` r
Plot2<-make_plot_gwas_catalog(dat=Dat,efo_id =EFO$efo_id,trait="glioma")
Plot2
```

!["README-example1\_gcplot1.png"](/man/figures/README-example1_gcplot1.png)

Each data point represents the Z score for glioma risk for a single SNP (scaled to reflect the reported effect allele in the GWAS catalog). The Y and X axes represent the Z scores in the test and GWAS catalog datasets, respectively. For most SNPs, the allele associated with higher risk in the GWAS catalog is associated with lower risk in the test dataset. We call these discrepancies "effect size conflicts" and their presence can be interpreted as evidence for an effect allele metadata error. However, when comparing datasets, it's important to make allowance for chance deviations in effect direction, especially for test datasets generated in small sample sizes. For this reason, effect size conflicts are labelled as high if the two-sided P value for the Z score is ≤0.0001 and as moderate if \>0.0001 (this is a pragmatic cutoff). When comparing datasets, one should also consider the number of SNPs. Effect size conflicts are more likely to reflect a metadata error when they are systematic across a large number of SNPs. In this glioma example, effect size conflicts are flagged across the vast majority of a reasonably large number of SNPs, which is strong evidence for an effect allele metadata error. This is consistent with the effect allele frequency conflicts flagged in the previous plot. In this particular example, the non-effect allele column was indeed mis-labelled as the effect allele.

We can also make a plot comparing effect allele frequency between the test dataset and the GWAS catalog, which we show in the next example.

``` r
Plot3<-make_plot_gwas_catalog(dat=Dat,plot_type="plot_eaf",efo=unique(Dat$efo),trait=unique(Dat$outcome))
Plot3
```

!["README-example1\_gcplot2.png"](/man/figures/README-example1_gcplot2.png)

We see an inverse correlation in reported effect allele frequency (EAF) between the test dataset and the GWAS catalog in European ancestry studies, which confirms the metadata error identified in the previous plots. In the absence of effect allele metadata errors, and assuming the test dataset and GWAS catalog are matched on ancestry, the correlation in allele frequency should be positive. The reported effect allele frequency in the test dataset is opposite to that observed in the GWAS catalog - e.g. for the top left red data point the effect allele is the major allele (frequency close to 0.8) in the test dataset but is the minor allele (frequency of 0.2) in the GWAS catalog. We call these discrepancies EAF conflicts. EAF conflicts are labelled as moderate if EAF is close to 0.5 (i.e. 0.4 to 0.6) and as high if \<0.4 or \>0.6. This makes allowance for chance deviations in allele frequency around 0.5. When making comparisons with the GWAS catalog it's important to consider whether the datasets are matched on ancestry. This consideration does not, however, apply for comparisons with the customised 1000 genomes reference dataset (see [step 2](#step2) above).

## Step 4. Check for effect sizes errors in the glioma GWAS

We next compare the expected and reported effect sizes. In this example we include two groups of SNPs: those associated with glioma in the GWAS catalog ("GWAS catalog associations") and those associated with glioma in the test dataset ("test GWAS top hits"). Typically these two sets of SNPs will strongly overlap but this is not necessarily always the case (and lack of overlap is itself a sign of potential problems). First, we make a list of SNPs corresponding to the GWAS catalog associations. Then we use the extract\_snps() function to extract those SNPs from the test dataset. We also set the argument "get\_sig\_snps" to TRUE, which tells the function to additionally extract GWAS significant SNPs from the test dataset (default p value is 5e-8). Alternatively to the extract\_snps() function, which does not work on Windows machines, you could use read.table() to read in your GWAS results file and then extract the two sets of SNPs. We then generate the expected effect sizes. Since the reported effect sizes correspond to log odds ratios, we use the predict\_lnor\_sh() function. This function was developed by [Sean Harrison](https://seanharrisonblog.com/2020/04/11/estimating-an-odds-ratio-from-a-gwas-only-reporting-the-p-value)

``` r
File<-system.file("extdata", "glioma_test_dat.txt", package = "CheckSumStats")
snplist<-make_snplist(efo_id=EFO$efo_id,trait="glioma",ref1000G_superpops=FALSE)
gli<-extract_snps(snplist=snplist,path_to_target_file=File,path_to_target_file_sep="\t",get_sig_snps=TRUE, p_val_col_number=7)
Dat<-format_data(dat=gli,outcome="Glioma",population="European",pmid=22886559,study="GliomaScan",ncase="cases",ncontrol="controls",rsid="Locus",effect_allele="Allele1",other_allele="Allele2",or="OR",or_lci="OR_95._CI_l",or_uci="OR_95._CI_u",eaf="eaf.controls",p="p",efo="glioma")
Pred<-predict_lnor_sh(dat=Dat)
Plot4<-make_plot_pred_effect(dat=Pred)
Plot4
```

!["README-example1\_predplot1.png"](/man/figures/README-example1_predplot1.png)

The plot shows a strong positive correlation between the expected and reported effect sizes, an intercept close to zero and a slope that is close to 1. This is reasonably close to what we'd expect to see in the absence of major analytical issues or errors in the summary data. Genetic association results for [arachidonic acid](#ara), [basal cell carcinoma](#bcc) and [colorectal cancer](#crc) provide examples where reported and expected effect sizes differ quite substantially.

Note that the predict\_lnor\_sh can be quite slow, so you may want to clump your results prior to using it, especially if you have \>100 SNPs. Below is how you would clump your results using the ieugwasr package.

``` r
Clump<-ieugwasr::ld_clump(clump_r2 = 0.01,clump_p=1e-8,dplyr::tibble(rsid=Dat$rsid, pval=Dat$p, id=Dat$id),pop="EUR")
Dat<-Dat[Dat$rsid %in% Clump$rsid,]
Pred<-predict_lnor_sh(dat=Dat)
Plot4<-make_plot_pred_effect(dat=Pred)
```

We can also plot the relative bias, i.e. the percentage deviation of the expected from the reported effect size.

``` r
Plot5<-make_plot_pred_effect(dat=Pred,bias=TRUE)
Plot5
```

!["README-example1\_predplot2.png"](/man/figures/README-example1_predplot2.png)

Overall the relative bias seems small and mostly varies from -10.9% to -13.8%, which seems reasonable. Given that genetic effect sizes tend to be small, a relative bias of 10% will be very small on an absolute scale (e.g. scaling an odds ratio of 1.10 up by 10% is 1.11).

## Step 5. Check that the top hits in the glioma test dataset are reported in the GWAS catalog

Next we check that the "top hits" for glioma in the test dataset are reported in the GWAS catalog. We define top hits as SNP-trait associations with P values \< 5e-8, a conventional threshold for statistical significance in GWAS. First we extract the top hits, using the extract\_sig\_snps() function but, alternatively, you could use the read.table() function to read in your entire dataset and then extract the top hits. If the top hits are not reported in the GWAS catalog this could be a sign of false positives in the test dataset, which in turn could be a sign of technical problems (such as failure to exclude low quality variants), although there may be alternative explanations. In the example below, we search the GWAS catalog for all genetic associations for glioma that are within 50,000 base pairs of the top hits in the test dataset.

``` r
File<-system.file("extdata", "glioma_test_dat.txt", package = "CheckSumStats")
gli<-extract_sig_snps(path_to_target_file=File,p_val_col_number=7)
Dat<-format_data(dat=gli,outcome="Glioma",population="European",pmid=22886559,study="GliomaScan",ncase="cases",ncontrol="controls",rsid="Locus",effect_allele="Allele1",other_allele="Allele2",or="OR",or_lci="OR_95._CI_l",or_uci="OR_95._CI_u",eaf="eaf.controls",p="p",efo="glioma")
gc_list<-find_hits_in_gwas_catalog(gwas_hits=Dat$rsid,efo_id=EFO$efo_id,distance_threshold=50000) 
gc_list
#> $not_in_gc
#> character(0)
#>
#> $in_gc
#> [1] "rs2736100"  "rs2853676"  "rs10120688" "rs1063192"  "rs1412829" 
#> [6] "rs2151280"  "rs2157719"  "rs7049105"  "rs4977756"  "rs6010620" 
#> [11] "rs6089953" 
```

All the top hits for glioma in the test dataset are either associated with glioma in the GWAS cataog or are in close physical proximity to a reported association for glioma (see $in\_gc). See [Example 3](#example3_notingc) for an example where most of the test dataset top hits are not reported in the GWAS catalog.

## Step 6. Check whether the reported P values correspond to the reported effect sizes in the glioma dataset

Next we generate some ZZ plots, in order to flag SNPs with P values that don't coincide with their reported effect sizes. The zz\_plot() function compares Zp scores (inferred from the reported P values) to Zb scores (inferred from the reported effect size and standard error). We see a very strong agreement between the Zb and Zp scores.

``` r
Plot6<-zz_plot(dat=Dat)
Plot6
```

!["example1\_zzplot.png"](/man/figures/README-example1_zzplot.png)

## Step 7. Combine all plots into a single report for the glioma GWAS

Next we combine all the plots into a single report.

``` r
Plot_list2<-ls()[grep("Plot[0-9]",ls())] 
Plot_list<-lapply(1:length(Plot_list2),FUN=function(x) eval(parse(text=Plot_list2[x])))
combine_plots(Plot_list=Plot_list,out_file="qc_report.png")
```

!["qc\_report.png"](/man/figures/README-qc_report.png)

# <a id="ara"></a> Example 2. Check the summary and metadata from a genome-wide association study of arachidonic acid.

In this example we use the package to check the summary and metadata from a genome-wide association study (GWAS) of arachidonic acid that has not gone through standad post-GWAS QC (e.g. with low quality or unreliable genetic variants excluded).

## 2.1. Extract and format the summary data for arachidonic acid

``` r
library(CheckSumStats)
EFO<-get_efo(trait="arachidonic acid measurement")
EFO

snplist<-make_snplist(efo_id=EFO$efo_id,trait="Plasma omega-6 polyunsaturated fatty acid levels (arachidonic acid)",ref1000G_superpops=TRUE)
File<-system.file("extdata", "ara_test_dat.txt", package = "CheckSumStats")
ara<-extract_snps(snplist=snplist,path_to_target_file=File)
Dat<-format_data(dat=ara,outcome="arachidonic acid",population="European",pmid=24823311,study="CHARGE",ncontrol="n",UKbiobank=FALSE,rsid="snp",effect_allele="effect_allele",other_allele="other_allele",beta="beta",se="se",eaf="effect_allele_freq",p="p")
```

## 2.2 Check allele frequency metadata for arachidonic acid GWAS

``` r
Plot1<-make_plot_maf(ref_1000G=c("AFR","AMR","EAS","EUR","SAS","ALL"),target_dat=Dat)
Plot1
```

!["example3\_mafplot.png"](/man/figures/README-example3_mafplot.png)

For the vast majority of SNPs, allele frequencies are compatible between the test dataset and 1000 genomes superpopulations. This indicates that the reported effect allele frequency column corresponds to the reported effect allele.

## 2.3. Check effect allele metadata for arachidonic acid

``` r
Plot2<-make_plot_gwas_catalog(dat=Dat,efo_id=EFO$efo_id,trait="Plasma omega-6 polyunsaturated fatty acid levels (arachidonic acid)",beta="beta",se="se",Title = "Comparison of associations in the GWAS catalog to the test dataset")
Plot2
```

!["example3\_gcplot1.png"](/man/figures/README-example3_gcplot1.png)

Most SNPs appear to have concordant effect sizes between the test dataset and the GWAS catalog. Although there are a few SNPs with effect sizes in opposite directions, the Z scores for these SNPs are small and therefore compatible with chance deviations. This suggests that the reported effect allele is correct.

## 2.4. Check for errors in the reported effect sizes in the arachidonic acid GWAS

Next, we check the summary data for possible effect size errors. For this step, we extract summary data for "top hits" in the test dataset.

``` r
File<-system.file("extdata", "ara_test_dat.txt", package = "CheckSumStats")
ara<-extract_sig_snps(path_to_target_file=File,p_val_col_number=7)
Dat<-format_data(dat=ara,outcome="arachidonic acid",population="European",pmid=24823311,study="CHARGE",ncontrol="n",UKbiobank=FALSE,rsid="snp",effect_allele="effect_allele",other_allele="other_allele",beta="beta",se="se",eaf="effect_allele_freq",p="p")
dim(Dat)
#> [1] 1064   19
```

1064 SNPs were extracted. It's useful to clump the results to ensure independence amongst SNPs and to speed up the QC checks to come. We call the ieugwasr package to perform the clumping.

``` r

Clump<-ieugwasr::ld_clump(clump_r2 = 0.01,clump_p=1e-8,dplyr::tibble(rsid=Dat$rsid, pval=Dat$p, id=Dat$id),pop="EUR")
#> Please look at vignettes for options on running this locally if you need to run many instances of this command.
#> Clumping , 1064 variants, using EUR population reference
#> Removing 969 of 1064 variants due to LD with other variants or absence from LD reference panel
Dat<-Dat[Dat$rsid %in% Clump$rsid,]
```

Now we generate expected effect sizes and plot these against the reported effect size.

``` r
Dat<-predict_beta_sd(dat=Dat)
Plot3<-make_plot_pred_effect(dat=Dat,pred_beta = "beta_sd",pred_beta_se="se_sd",beta="beta",se="se")
Plot3
```

!["example3\_predplot1.png"](/man/figures/README-example3_predplot1.png)

We see a slope of 0.548 and non-linear correlation pattern, indicating that the SNPs have unusual effect sizes. This patterm of results is compatible with potential problems in the post-GWAS cleaning of the summary statistics. Discrepancies between reported and expected effect sizes can also arise for other reasons: [Example 4](#bcc) and [Example 5](#crc)

We can also plot the relative bias - the relative deviation of the reported from the expected effect sizes.

``` r
Plot4<-make_plot_pred_effect(dat=Dat,pred_beta = "beta_sd",pred_beta_se="se_sd",beta="beta",se="se",bias=TRUE)
Plot4
```

!["example3\_predplot2.png"](/man/figures/README-example3_predplot2.png)

The SNPs with the most bias tend to have lower minor allele frequencies, perhaps reflecting problems with the post-GWAS cleaning of the results file.

## 2.5. Check that the top hits in the arachidonic acid test dataset are reported in the GWAS catalog

# <a id="example3_notingc"></a>

``` r
gc_list<-find_hits_in_gwas_catalog(gwas_hits=Dat$rsid,efo_id=EFO$efo_id,trait="Plasma omega-6 polyunsaturated fatty acid levels (arachidonic acid)",distance_threshold=50000) 
gc_list
#> $not_in_gc
#>  [1] "rs10026364" "rs10488885" "rs10819512" "rs10938476" "rs10986188"
#>  [6] "rs11726352" "rs12238732" "rs12416578" "rs12456907" "rs12639648"
#> [11] "rs12888839" "rs12894905" "rs12955978" "rs13128117" "rs13268288"
#> [16] "rs13381393" "rs16940996" "rs16943026" "rs16951711" "rs16974991"
#> [21] "rs17060495" "rs17568699" "rs1887892"  "rs2192242"  "rs2826490" 
#> [26] "rs3088245"  "rs340480"   "rs4359352"  "rs4818759"  "rs7080747" 
#> [31] "rs7093714"  "rs7226534"  "rs7231821"  "rs727887"   "rs7292052" 
#> [36] "rs7456249"  "rs7690731"  "rs7778698"  "rs7904736"  "rs12747494"
#> [41] "rs10788947" "rs10798816" "rs957129"   "rs11578575" "rs7546429" 
#> [46] "rs12037648" "rs6658106"  "rs12623171" "rs6750701"  "rs13386900"
#> [51] "rs13314643" "rs13325952" "rs16851412" "rs11917725" "rs17077488"
#> [56] "rs11958171" "rs279412"   "rs5745104"  "rs13178241" "rs10040997"
#> [61] "rs1432985"  "rs781980"   "rs9275354"  "rs13191761" "rs9356335" 
#> [66] "rs12523734" "rs6456902"  "rs11752402" "rs2294281"  "rs9375065" 
#> [71] "rs12285167" "rs487023"   "rs7933136"  "rs2903922"  "rs930786"  
#> [76] "rs760306"   "rs11824358" "rs890455"   "rs10830946" "rs259874"  
#> [81] "rs11833369" "rs7970058"  "rs4931549"  "rs11147144" "rs7137292" 
#> [86] "rs12297743" "rs11107024" "rs2653765"  "rs16959795" "rs16985879"
#> [91] "rs4614987"  "rs6060682" 
#> 
#> $in_gc
#> [1] "rs174528" "rs472031" "rs1741"

Plot5<-make_plot_gwas_catalog(dat=Dat,efo_id=EFO$efo_id,trait="Plasma omega-6 polyunsaturated fatty acid levels (arachidonic acid)",force_all_trait_study_hits=TRUE,beta="beta",se="se",Title = "Comparison of top hits in test dataset to GWAS catalog")
Plot5
```

!["example3\_gcplot3.png"](/man/figures/README-example3_gcplot3.png)

92 of 95 top hits in the test dataset do not overlap with associations for arachidonic acid in the GWAS catalog. Taken together with the non-linear relationship between the expected and reported effect sizes, this suggests a large number of false positives, which may in turn reflect problems with the post-GWAS cleaning of the summary statistics. Correspondence with the data provider confirmed that the GWAS summary statistics had not gone through post GWAS filtering of low quality variants (e.g. exclusion of SNPs with low minor allele frequency or low imputation r2 scores). Once we obtained a cleaned dataset (with low quality SNPs excluded), the aforementioned discrepancies were resolved.

## 2.6 Check whether the reported P values correspond to the reported effect sizes (arachidonic acid example)

Finally, we check whether the reported P values correspond to the reported effect sizes.

``` r
Plot6<-zz_plot(dat=Dat,beta="beta",se="se")
Plot6
```

!["example3\_zzplot.png"](/man/figures/README-example3_zzplot.png)

We see a very close concordance between the reported P-values and reported effect sizes.

## 2.7 Combine all plots into a single report

Let's combine all the key figures into a single report

``` r
Plot_list2<-c("Plot1","Plot2","Plot3","Plot4","Plot5","Plot6")
Plot_list<-lapply(1:length(Plot_list2),FUN=function(x) eval(parse(text=Plot_list2[x])))
combine_plots(Plot_list=Plot_list,out_file="~/qc_report2.png")
```

!["qc\_report.png"](/man/figures/README-qc_report2.png)

# Other examples

## <a id="fasting_glucose></a> An allele frequency metadata error in a GWAS of fasting glucose.

In this example we use the package to check the allele frequency metadata from a genome-wide association study of fasting glucose. In this example, minor allele frequency was incorrectly assumed to be the same as effect allele frequency.

``` r
library(CheckSumStats)
EFO<-get_efo(trait="fasting glucose")
snplist<-make_snplist(efo_id=EFO$efo_id,trait="fasting blood glucose",ref1000G_superpops=TRUE)
glu <- ieugwasr::associations(id="ebi-a-GCST005186", variants=snplist,proxies=0)  
Dat<-format_data(dat=glu,outcome="Fasting glucose",population="European",pmid=22581228,study="",ncontrol="n",rsid="rsid",effect_allele="ea",other_allele="nea",beta="beta",se="se",eaf="eaf",p="p")
Plot1<-make_plot_maf(ref_1000G=c("AFR","AMR","EAS","EUR","SAS","ALL"),target_dat=Dat)
Plot1
```

!["example2.png"](/man/figures/README-example2.png)

Each red data point corresponds to an allele frequency conflict and is identified for approximately half of the SNPs. This pattern occurs when reported effect allele frequency corresponds to minor allele frequency and the minor allele has not been specifically modelled as the effect allele.

## <a id="bcc"></a> Effect size scale issues in a genome-wide association study of basal cell carcinoma

One of the limitations of using GWAS results from online platforms is that the scale of the effect sizes may not always be clear. This can hamper comparison of results from different studies. For example, for some studies conducted in large population biobanks, disease case-control status has been assessed using linear mixed models, when most previously conducted case-control GWAS have employed logistic regression models. One way to test that the effect sizes correspond to log odds ratios is to use the predict\_lnor\_sh function, which generates expected log odds ratios based on Z scores, allele frequency and sample size. These can then be compared to the reported effect sizes. Disagreements between the two could suggest differences in the underlying statistical models used to generate the results. In the next example, we compare the reported and expected effect sizes from a GWAS of basal cell carcinoma conducted in UK Biobank. The effect sizes were generated in a linear mixed model, with case-control status (controls coded 1 and cases coded 2) regressed on genotype.

``` r
library(CheckSumStats)
EFO<-get_efo(trait="basal cell carcinoma")
snplist<-make_snplist(efo_id = EFO$efo_id)
bcc <- ieugwasr::associations(id="ukb-b-8837", variants=snplist,proxies=0)  
dat<-format_data(dat=bcc,outcome="Basal cell carcinoma",population="European",ncase=4290,ncontrol=458643,study="UKB",rsid="rsid",effect_allele="ea",other_allele="nea",lnor="beta",lnor_se="se",eaf="eaf",p="p",efo_id = EFO$efo_id)
Clump<-ieugwasr::ld_clump(clump_r2 = 0.01,clump_p=1e-8,dplyr::tibble(rsid=dat$rsid, pval=dat$p, id=dat$id),pop="EUR")
dat2<-dat[dat$rsid %in% Clump$rsid,]
Pred<-predict_lnor_sh(dat=dat2)
lm(Pred$lnor_pred ~ Pred$lnor)$coefficients
#> (Intercept)    Pred$lnor 
#> 0.01590922 110.47612861 

Plot4<-make_plot_pred_effect(dat=data.frame(Pred))
Plot4
```

!["example4\_1.png"](/man/figures/README-example4_1.png)

The slope of the relationship between the expected and reported effect sizes is 110, when we expect it to be 1. This discrepancy arises because the reported effect sizes correspond to absolute changes in risk, whereas the expected effect sizes correspond to log odds ratios. To transform the reported effect sizes to log odds ratios, we can use the transform\_betas function. After applying this transformation, we see that the regression slope is very close to 1.

``` r
dat2<-transform_betas(dat=dat2,effect="lnor",effect.se="lnor_se")
Pred<-predict_lnor_sh(dat=dat2)
lm(Pred$lnor_pred ~ Pred$lnor)
#> (Intercept)   Pred$lnor 
#> 0.01590922  1.01429487 
Plot4<-make_plot_pred_effect(dat=data.frame(Pred))
Plot4
```

!["example4\_2.png"](/man/figures/README-example4_2.png)

# <a id="crc"></a> Sample size errors in a genome-wide association study of colorectal cancer

Discrepancies between expected and reported effect sizes can also arise from errors in reported allele frequency or SNP-level sample sizes. In the next example, we assess results from a GWAS meta-analysis of colorectal cancer, where the number of samples contributing to the analysis varied across SNPs.

``` r
File<-system.file("extdata", "crc_test_dat.txt", package = "CheckSumStats")
crc<-read.table(File,sep="\t",head=TRUE,stringsAsFactors=FALSE)
dat<-format_data(dat=crc,outcome="Colorectal cancer",population="East Asian",ncase=23572,ncontrol=48700,study="ACCC",rsid="rsid",effect_allele="Allele1",other_allele="Allele2",lnor="Effect",lnor_se="StdErr",eaf="Freq1",p="P.value")
Pred<-predict_lnor_sh(dat=dat)
lm(Pred$lnor_pred ~ Pred$lnor)
#> (Intercept)   Pred$lnor 
#> 0.002193062 0.231316576 
```

!["example5.png"](/man/figures/README-example5.png)

The slope of the relationship between the expected and reported effect sizes is 0.23 (the expected slope is 1). The shape of the relationship between the two sets of effect sizes is also non-linear. The discrepancy has arisen due to incorrect assumptions about sample size across SNPs. The expected effect sizes were generated assuming 23572 cases and 48700 controls across all SNPs. However, in this example the number of contributing studies varies quite substantially across SNPs:

``` r
summary(dat$Nstudies)
#> Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
#> 1.00   11.75   14.00   11.69   14.00   15.00 
```

We can see that the number of studies contributing to each SNP analysis varies from a minimum of 1 to 15. For 25% of the SNPs, analyses were based on \<12 studies. This implies that the number of cases and controls that contributed to the analyses varied across SNPs (in this example we lacked direct information on SNP-level sample sizes and only had information on the maximum reported sample size).

``` r
Pos<-Pred$Nstudies>=14
lm(Pred$lnor_pred[Pos] ~ Pred$lnor[Pos])
#> (Intercept) Pred$lnor[Pos] 
#> 6.634408e-05   8.379027e-01 
Pos<-Pred$Nstudies<14
lm(Pred$lnor_pred[Pos] ~ Pred$lnor[Pos])
#> (Intercept) Pred$lnor[Pos] 
#> 4.587284e-05   2.140147e-01 
```

For the subset of SNP results generated in analyses of ≥14 studies (and thus with a sample size closer to the assumed sample size) the slope was much closer to expectation (slope=0.84). In contrast, for results generated in \<14 studies, the slope was 0.21.

# Acknowledgements

We gratefully acknowledge the help of Ramiro Magno for their help and advice with the gwasrapidd package.

CheckSumStats greatfully acknowledges the following packages:

    gwasrapidd
    ggplot2
    grid
    gridExtra
    cowplot
    grDevices
    ieugwasr
    knitr
    biomaRt
    purrr
    dplyr
    tibble
    magrittr
    curl
    plyr
    utils
    stats
