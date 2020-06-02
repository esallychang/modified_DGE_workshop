---
title: "Gene-level differential expression analysis with DESeq2"
author: "Meeta Mistry, Radhika Khetani, Mary Piper"
date: "October 13th, 2017"
---

Approximate time: 60 minutes

## Learning Objectives 

* LFC shrinkage 
* Gene-level filtering?
* Building results tables for comparison of different sample classes
* Summarizing significant differentially expressed genes for each comparison

## MOV10 Differential Expression Analysis: Control versus Overexpression

We have three sample classes so we can make three possible pairwise comparisons:

1. Control vs. Mov10 overexpression
2. Control vs. Mov10 knockdown
3. Mov10 knockdown vs. Mov10 overexpression

**We are really only interested in #1 and #2 from above**. Using the design formula we provided `~ sampletype`, indicating that this is our main factor of interest.

### Creating contrasts

To indicate to DESeq2 the two groups we want to compare, we can use **contrasts**. Contrasts are then provided to DESeq2 to perform differential expression testing using the Wald test. Contrasts can be provided to DESeq2 a couple of different ways:

1. Do nothing. Automatically DESeq2 will use the base factor level of the condition of interest as the base for statistical testing. The base level is chosen based on alphabetical order of the levels.
2. In the `results()` function you can specify the comparison of interest, and the levels to compare. The level given last is the base level for the comparison. The syntax is given below:
	
```r
	# DO NOT RUN!
	contrast <- c("condition", "level_to_compare", "base_level")
	results(dds, contrast = contrast, alpha = alpha_threshold)
```

>
> **NOTE:** The Wald test can also be used with **continuous variables**. If the variable of interest provided in the design formula is continuous-valued, then the reported log2 fold change is per unit of change of that variable.


### Building the results table

To build our results table we will use the `results()` function. To tell DESeq2 which groups we wish to compare, we supply the contrasts we would like to make using the`contrast` argument. 

```r
## Define contrasts, extract results table, and shrink the log2 fold changes

contrast_oe <- c("sampletype", "MOV10_overexpression", "control")

res_tableOE <- results(dds, contrast=contrast_oe, alpha = 0.05)
```
Above we provided the bare minimum for the `results()` function. Take a look at the help manual to see the other arguments that we can modify:

```r
?results
```

* **Independent filtering**: We are including the `alpha` argument and setting it to 0.05. This is the significance cutoff used for optimizing the independent filtering (by default it is set to 0.1). If the adjusted p-value cutoff (FDR) will be a value other than 0.1 (for our final list of significant genes), `alpha` should be set to that value. There is also an argument to turn off the filtering off by setting `independentFiltering = F`.

> **What is indepdendent filtering?** This is a low mean threshold that is empirically determined from your data, in which the fraction of significant genes can be increased by reducing the number of genes that are considered in teh muliple testing.
>
>  <img src="../img/indp_filt.png" width="600">
> 
> *Image courtesy of [slideshare presentation](https://www.slideshare.net/joachimjacob/5rna-seqpart5detecting-differentialexpression) from Joachim Jacob, 2014.*

* **Cooks cutoff**: We can also turn of the filtering to remove extreme outlier genes with `cooksCutoff`
* **Multiple correction**: In DESeq2, the p-values attained by the Wald test are corrected for multiple testing using the Benjamini and Hochberg method by default. There are options to use other methods using the `pAdjustMethod` argument

### Results exploration

The results table looks very much like a dataframe and in many ways it can be treated like one (i.e when accessing/subsetting data). However, it is important to recognize that it is actually stored in a `DESeqResults` object. When we start visualizing our data, this information will be helpful. 

```r
class(res_tableOE)
```

Let's go through some of the columns in the results table to get a better idea of what we are looking at. To extract information regarding the meaning of each column we can use `mcols()`:

```r
mcols(res_tableOE, use.names=T)
```

* `baseMean`: mean of normalized counts for all samples
* `log2FoldChange`: log2 fold change
* `lfcSE`: standard error
* `stat`: Wald statistic
* `pvalue`: Wald test p-value
* `padj`: BH adjusted p-values
 

Now let's take a look at what information is stored in the results:

```r
res_tableOE %>% data.frame() %>% View()
```

```
log2 fold change (MAP): sampletype MOV10_overexpression vs control 
Wald test p-value: sampletype MOV10_overexpression vs control 
DataFrame with 57914 rows and 6 columns
               		baseMean	log2FoldChange	lfcSE		stat		pvalue		padj
              		<numeric>	<numeric>	<numeric>	<numeric>	<numeric>	<numeric>
ENSG00000000003		3.53E+03	-0.427190489	0.0755347	-5.65604739	1.55E-08	4.47E-07
ENSG00000000005		2.62E+01	0.016159765	0.23735203	0.06584098	9.48E-01	9.74E-01
ENSG00000000419		1.48E+03	0.362663551	0.10761742	3.36995355	7.52E-04	4.91E-03
ENSG00000000457		5.19E+02	0.219135591	0.09768842	2.24476439	2.48E-02	8.21E-02
ENSG00000000460		1.16E+03	-0.261603812	0.07912962	-3.30661411	9.44E-04	5.92E-03
...			...		...		...		...		...		...
```

**The order of the names in the contrast determines the direction of fold change that is reported.** The name provided in the second element is the level that is used as baseline. So for example, if we observe a log2 fold change of -2 this would mean the gene expression is lower in Mov10_oe relative to the control. However, these estimates do not account for the large dispersion we observe with low read counts. To avoid this, the **log2 fold changes calculated by the model need to be adjusted**. 

Although the fold changes provided is important to know, ultimately the **p-adjusted values should be used to determine significant genes**. The significant genes can be output for visualization and/or functional analysis.


> **NOTE: on p-values set to NA**
> > 
> 1. If within a row, all samples have zero counts, the baseMean column will be zero, and the log2 fold change estimates, p-value and adjusted p-value will all be set to NA.
> 2. If a row contains a sample with an extreme count outlier then the p-value and adjusted p-value will be set to NA. These outlier counts are detected by Cook’s distance. 
> 3. If a row is filtered by automatic independent filtering, for having a low mean normalized count, then only the adjusted p-value will be set to NA. 

### Shrunken log2 foldchanges (LFC)

To generate more accurate log2 foldchange estimates, DESeq2 allows for the **shrinkage of the LFC estimates toward zero** when the information for a gene is low, which could include:

- Low counts
- High dispersion values

As with the shrinkage of dispersion estimates, LFC shrinkage uses **information from all genes** to generate more accurate estimates. Specifically, the distribution of LFC estimates for all genes is used (as a prior) to shrink the LFC estimates of genes with little information or high dispersion toward more likely (lower) LFC estimates. 

<img src="../img/deseq2_shrunken_lfc.png" width="500">

*Illustration taken from the [DESeq2 paper](https://genomebiology.biomedcentral.com/articles/10.1186/s13059-014-0550-8).*

For example, in the figure above, the green gene and purple gene have the same mean values for the two sample groups (C57BL/6J and DBA/2J), but the green gene has little variation while the purple gene has high levels of variation. For the green gene with low variation, the **unshrunken LFC estimate** (vertex of the green **solid line**) is very similar to the shrunken LFC estimate (vertex of the green dotted line), but the LFC estimates for the purple gene are quite different due to the high dispersion. So even though two genes can have similar normalized count values, they can have differing degrees of LFC shrinkage. Notice the **LFC estimates are shrunken toward the prior (black solid line)**.

In the most recent versions of DESeq2, the shrinkage of LFC estimates is **not performed by default**. This means that the log2 foldchanges would be the same as those calculated by:

```r
log2 (normalized_counts_group1 / normalized_counts_group2)
```

To generate the shrunken log2 fold change estimates, you have to run an additional step on your results object (that we will create below) with the function `lfcShrink()`.

```r
## Save the unshrunken results to compare
res_tableOE_unshrunken <- res_tableOE

# Apply fold change shrinkage
res_tableOE <- lfcShrink(dds, contrast=contrast_oe, res=res_tableOE)
```

> **NOTE: Shrinking the log2 fold changes will not change the total number of genes that are identified as significantly differentially expressed.** The shrinkage of fold change is to help with downstream assessment of results. For example, if you wanted to subset your significant genes based on fold change for further evaluation, you may want to use shruken values. Additionally, for functional analysis tools such as GSEA which require fold change values as input you would want to provide shrunken values.


### MA Plot

A plot that can be useful to exploring our results is the MA plot. The MA plot shows the mean of the normalized counts versus the log2 foldchanges for all genes tested. The genes that are significantly DE are colored to be easily identified. This is also a great way to illustrate the effect of LFC shrinkage. The DESeq2 package offers a simple function to generate an MA plot. 

**Let's start with the unshrunken results:**

```r
plotMA(res_tableOE_unshrunken, ylim=c(-2,2))
```

<img src="../img/maplot_unshrunken.png" width="600">

**And now the shrunken results:**

```r
plotMA(res_tableOE, ylim=c(-2,2))
```

<img src="../img/MA_plot.png" width="600">

In addition to the comparison described above, this plot allows us to evaluate the magnitude of fold changes and how they are distributed relative to mean expression. Generally, we would expect to see significant genes across the full range of expression levels. 


## MOV10 Differential Expression Analysis: Control versus Knockdown

Now that we have results for the overexpression results, let's do the same for the **Control vs. Knockdown samples**. Use contrasts in the `results()` to extract a results table and store that to a variable called `res_tableKD`.  

```r
## Define contrasts, extract results table and shrink log2 fold changes
contrast_kd <-  c("sampletype", "MOV10_knockdown", "control")

res_tableKD <- results(dds, contrast=contrast_kd, alpha = 0.05)

res_tableKD <- lfcShrink(dds, contrast=contrast_kd, res=res_tableKD)
```

Take a quick peek at the results table containing Wald test statistics for the Control-Knockdown comparison we are interested in and make sure that format is similar to what we observed with the OE.

## Summarizing results

To summarize the results table, a handy function in DESeq2 is `summary()`. Confusingly it has the same name as the function used to inspect data frames. This function when called with a DESeq results table as input, will summarize the results using the alpha threshold: FDR < 0.05 (padj/FDR is used even though the output says `p-value < 0.05`). Let's start with the OE vs control results:

```r
## Summarize results
summary(res_tableOE, alpha = 0.05)
```

In addition to the number of genes up- and down-regulated at the default threshold, **the function also reports the number of genes that were tested (genes with non-zero total read count), and the number of genes not included in multiple test correction due to a low mean count**.


## Extracting significant differentially expressed genes

Let's first create variables that contain our threshold criteria:

```r
### Set thresholds
padj.cutoff <- 0.05
```

We can easily subset the results table to only include those that are significant using the `filter()` function, but first we will convert the results table into a tibble:

```r
res_tableOE_tb <- res_tableOE %>%
  data.frame() %>%
  rownames_to_column(var="gene") %>% 
  as_tibble()
```

Now we can subset that table to only keep the significant genes using our pre-defined thresholds:

```r
sigOE <- res_tableOE_tb %>%
        filter(padj < padj.cutoff)
```

```r
sigOE
```


Using the same p-adjusted threshold as above (`padj.cutoff < 0.05`), subset `res_tableKD` to report the number of genes that are up- and down-regulated in Mov10_knockdown compared to control.

```r

res_tableKD_tb <- res_tableKD %>%
  data.frame() %>%
  rownames_to_column(var="gene") %>% 
  as_tibble()
  
sigKD <- res_tableKD_tb %>%
        filter(padj < padj.cutoff)
```

**How many genes are differentially expressed in the Knockdown compared to Control?** 
```r
sigKD
``` 

Now that we have extracted the significant results, we are ready for visualization!

> ### Adding a fold change threshold: 
> With large significant gene lists it can be hard to extract meaningful biological relevance. To help increase stringency, one can also **add a fold change threshold**.
> 
> For e.g., we can create a new threshold `lfc.cutoff` and set it to 0.58 (remember that we are working with log2 fold changes so this translates to an actual fold change of 1.5).
> 
> `lfc.cutoff <- 0.58`
> 
> `sigOE <- res_tableOE_tb %>% filter(padj < padj.cutoff & abs(log2FoldChange) > lfc.cutoff)`

> ### An alternative approach to add the fold change threshold:
> The `results()` function has an option to add a fold change threshold using the `lfcThrehsold` argument. This method is more statistically motivated, and is recommended when you want a more confident set of genes based on a certain fold-change. It actually performs a statistical test against the desired threshold, by performing a two-tailed test for log2 fold changes greater than the absolute value specified. The user can change the alternative hypothesis using `altHypothesis` and perform two one-tailed tests as well. **This is a more conservative approach, so expect to retrieve a much smaller set of genes!**
>
> Test this out using our data:
> 
> `results(dds, contrast = contrast_oe, alpha = 0.05, lfcThreshold = 0.58)`
>
> **How do the results differ? How many significant genes do we get using this approach?**


---
*This lesson has been developed by members of the teaching team at the [Harvard Chan Bioinformatics Core (HBC)](http://bioinformatics.sph.harvard.edu/). These are open access materials distributed under the terms of the [Creative Commons Attribution license](https://creativecommons.org/licenses/by/4.0/) (CC BY 4.0), which permits unrestricted use, distribution, and reproduction in any medium, provided the original author and source are credited.*

*Some materials and hands-on activities were adapted from [RNA-seq workflow](http://www.bioconductor.org/help/workflows/rnaseqGene/#de) on the Bioconductor website*

***