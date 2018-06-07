## Summarising and visualising multiple MAF files

Script summarising and visualising multiple [MAF](https://software.broadinstitute.org/software/igv/MutationAnnotationFormat) files. It produces ***summariseMAFs.html*** report and *[summariseMAFs.md](https://github.com/umccr/MAF-summary/tree/master/scripts/summariseMAFs.md)* markdown report.

<br>

Make sure to set the java max heap size to 2Gb to accomodate big gene tables written into excel spreadsheet using the xlsx R package

```r
##### Set the jave max heap size to 2Gb to accomodate big gene tables
options( java.parameters = "-Xmx2000m" )
```

<br>

### Load libraries

```r
suppressMessages(library(knitr))
suppressMessages(library(maftools))
suppressMessages(library(xlsx))
suppressMessages(library(optparse))
suppressMessages(library(DT))
```

## Read MAF files

Go to the MAF files directory, read the files and create directory for output files, if does not exist already

```r
##### Read MAF files and put associated info into a list
mafFiles <- gsub("\\s","", params$mafFiles)
mafFiles <-  unlist(strsplit(mafFiles, split=',', fixed=TRUE))
mafFiles <- paste(params$mafDir, mafFiles, sep="/")

mafInfo <- vector("list", length(mafFiles))

for ( i in 1:length(mafFiles) ) {
  mafInfo[[i]] = read.maf(maf = mafFiles[i], verbose = FALSE)
}
```

```r
##### Create directory for output files
outDir <- paste(params$mafDir, params$outDir, sep = "/")

if ( !file.exists(params$outDir) ){
  dir.create(outDir)
}
```

### Summarise MAF files - tables

This part will create set of tabs with interactive tables including *overall summary*, *samples summary* and *genes summary* (check the *[summariseMAFs.html](https://github.com/umccr/MAF-summary/tree/master/scripts/summariseMAFs.html)* example report).
 
### Overall summary

Generate tables  with basic information about each MAF file, including NCBI build, no. fo samples and genes, no. of different mutation types ( frameshift deletions, frameshift insertions, in-frame deletions, in-frame insertions, missense mutations, nonsense mutations, nonstop mutations, splice site mutations, translation start site mutations), as well as the total no. of mutations present in the MAF file. Individual tables present summary for corresponding datasets.

```r
cohorts.list <- gsub("\\s","", params$cohorts)
cohorts.list <- unlist(strsplit(cohorts.list, split=',', fixed=TRUE))

##### Write overall summary into a file
if ( !file.exists(paste(outDir, "MAF_summary.xlsx", sep = "/")) ){
  for ( i in 1:length(mafFiles) ) {
    write.xlsx(mafInfo[[i]]@summary, file=paste(outDir, "MAF_summary.xlsx", sep="/"), sheetName=cohorts.list[i], row.names=FALSE,  append=TRUE)
  }
}

##### Present a MAF file summary table in the html report
##### Create a list for htmlwidgets
widges.list <- htmltools::tagList()

for ( i in 1:length(mafFiles) ) {
  widges.list[[i]] <- DT::datatable(data = mafInfo[[i]]@summary, caption = paste0("MAF summary: ", cohorts.list[i]))
}

##### Print a list of htmlwidgets
widges.list
```

Additionally ceate an excel spreadsheet listing all fields (columns) in the individaul MAF files.

```r
##### Get all fields in MAF files
if ( !file.exists(paste(outDir, "MAF_fields.xlsx", sep = "/")) ){
  for ( i in 1:length(mafFiles) ) {
    write.xlsx(maftools::getFields(mafInfo[[i]]), file=paste(outDir, "MAF_fields.xlsx", sep="/"), sheetName=cohorts.list[i], row.names=FALSE,  append=TRUE, col.names=FALSE)
  }
}
```

### Samples summary

Create tables with samples summary. Each table contains per-sample information (rows) about no. of different types of mutations (columns), including frameshift deletions, frameshift insertions, in-frame deletions, in-frame insertions, missense mutations, nonsense mutations, nonstop mutations, splice site mutations, translation start site mutations, as well as the total no. of mutations present in the MAF file.


```r
##### Write samples summary into a file
if ( !file.exists(paste(outDir, "MAF_sample_summary.xlsx", sep = "/")) ) {

  for ( i in 1:length(mafFiles) ) {
    write.xlsx(maftools::getSampleSummary(mafInfo[[i]]), file=paste(outDir, "MAF_sample_summary.xlsx", sep="/"), sheetName=cohorts.list[i], row.names=FALSE,  append=TRUE)
  }
}

##### Present a sample table in the html report
##### Create a list for htmlwidgets
widges.list <- htmltools::tagList()

for ( i in 1:length(mafFiles) ) {
  widges.list[[i]] <- DT::datatable(data = maftools::getSampleSummary(mafInfo[[i]]), caption = paste0("Samples summary: ", cohorts.list[i]), filter = "top")
}

##### Print a list of htmlwidgets
widges.list
```

### Genes summary

Now present gene summary info. This part creates tables for individual cohorts with per-gene information (rows) about no. of different types of mutations (columns), including frameshift deletions, frameshift insertions, in-frame deletions, in-frame insertions, missense mutations, nonsense mutations, nonstop mutations, splice site mutations, translation start site mutations, as well as the total no. of mutations present in the MAF file. The last two columns contain the no. of samples with mutations/alterations in the corresponding gene.


```r
##### Write gene summary into a file
if ( !file.exists(paste(outDir, "MAF_gene_summary.xlsx", sep = "/")) ){
  for ( i in 1:length(mafFiles) ) {
    write.xlsx(maftools::getGeneSummary(mafInfo[[i]]), file=paste(outDir, "MAF_gene_summary.xlsx", sep="/"), sheetName=cohorts.list[i], row.names=FALSE,  append=TRUE)
  }
}

##### Present a gene table in the html report
##### Create a list for htmlwidgets
widges.list <- htmltools::tagList()

for ( i in 1:length(mafFiles) ) {
  widges.list[[i]] <- DT::datatable(data = maftools::getGeneSummary(mafInfo[[i]]), caption = paste0("Genes summary: ", cohorts.list[i]), filter = "top")
}

##### Print a list of htmlwidgets
widges.list
```


### Summarise MAF files - heatmaps

This part will create set of tabs with interactive heatmaps summarising samples (*samples summary* tab) and genes info (*genes summary* tab). Check the *[summariseMAFs.html](https://github.com/umccr/MAF-summary/tree/master/scripts/summariseMAFs.html)* report to see an example.

#### Samples summary

Generate an interactive heatmap to facilitate outlier samples detection. Rows and columns represent samples and mutation types, respectively. The colour scale from blue to yellow indicates low and high number of various mutations types, respectively, reported for corresponding samples. Samples are ordered by the number of mutations to facilitate identification of individuals with extreme mutation burden.

```r
suppressMessages(library(plotly))
suppressMessages(library(heatmaply))

##### Create a list for htmlwidgets
widges.list <- htmltools::tagList()

##### Display samples summary in a form of interactive heatmap
for ( i in 1:length(mafFiles) ) {

  sampleSummary <- data.frame(maftools::getSampleSummary(mafInfo[[i]]))
  rownames(sampleSummary) <-sampleSummary[,"Tumor_Sample_Barcode"]
  sampleSummary <- subset(sampleSummary, select=-c(Tumor_Sample_Barcode, total))

  ##### Generate interactive heatmap
  p <- heatmaply(sampleSummary, main = paste0("Samples summary: ", cohorts.list[i]), Rowv=NULL, Colv=NULL, scale="none", dendrogram="none", trace="none", hide_colorbar = FALSE, fontsize_row = 8, label_names=c("Sample","Mutation_type","Count")) %>%
  layout(width  = 900, height = 600, margin = list(l=150, r=10, b=150, t=50, pad=4), titlefont = list(size=16), xaxis = list(tickfont=list(size=10)), yaxis = list(tickfont=list(size=10)))

  ##### Add plot to the list for htmlwidgets
  widges.list[[i]] <- as_widget(ggplotly(p))

   ##### Save the heatmap as html (PLOTLY)
  htmlwidgets::saveWidget(as_widget(p), paste0(outDir, "/MAF_sample_summary_heatmap_", cohorts.list[i], ".html"), selfcontained = TRUE)

  ##### Plotly option
  #p <- plot_ly(x = colnames(sampleSummary), y = rownames(sampleSummary), z = as.matrix(sampleSummary), height = 600, type = "heatmap") %>%
  #layout(title = paste0("Samples summary: ", cohorts.list[i]), autosize = TRUE, margin = list(l=150, r=10, b=100, t=100, pad=4), showlegend = TRUE)

  #widges.list[[i]] <- ggplotly(p)
}
```

```r
##### Detach plotly package. Otherwise it clashes with other graphics devices
detach("package:heatmaply", unload=FALSE)
detach("package:plotly", unload=FALSE)

##### Print a list of htmlwidgets
widges.list
```
*Note, since html files are not supported by GitHub the follwoing plots are screenshots only*

![](summariseMAFs_files/figure-html/MAF_sample_summary_heatmap_TCGA-PAAD.png)<!-- -->

![](summariseMAFs_files/figure-html/MAF_sample_summary_heatmap_ICGC-PACA-AU.png)<!-- -->

![](summariseMAFs_files/figure-html/MAF_sample_summary_heatmap_ICGC-PACA-AU-additional.png)<!-- -->

![](summariseMAFs_files/figure-html/MAF_sample_summary_heatmap_ICGC-PACA-CA.png)<!-- -->

<br>

### Genes summary

This time generate an interactive heatmap to summmarise genes information. Rows and columns represent genes and mutation types, respectively. The colour scale from blue to yellow indicates low and high number of various mutations types, respectively, observed in corresponding genes. Genes are ordered by the number of reported mutations. The total number of mutations in individual genes, as well as the number of samples with mutations are also presented in the last three columns. Note, for transparency we show only the top 50 mutated genes.

```r
suppressMessages(library(plotly))
suppressMessages(library(heatmaply))

##### Create a list for htmlwidgets
widges.list <- htmltools::tagList()

##### Display genes summary in a form of interactive heatmap
for ( i in 1:length(mafFiles) ) {

  geneSummary <- data.frame(maftools::getGeneSummary(mafInfo[[i]])[1:50,])
  rownames(geneSummary) <-geneSummary[,"Hugo_Symbol"]
  geneSummary <- subset(geneSummary, select=-c(Hugo_Symbol))

  ##### Cluster table by genes
  #hr <- hclust(as.dist(dist(geneSummary, method="euclidean")), method="ward.D")

  ##### Generate interactive heatmap
  #p <- heatmaply(geneSummary, main = paste0("Genes  summary: ", cohorts.list[i]), Rowv=as.dendrogram(hr), Colv=NULL, scale="none", dendrogram="none", trace="none", hide_colorbar = TRUE, fontsize_row = 8, fontsize_col = 8)  %>%
  #layout(autosize = TRUE, width = 800, height = 800, margin = list(l=250, r=10, b=150, t=50, pad=4), showlegend = TRUE)

  ##### Generate interactive heatmap
  p <- heatmaply(geneSummary, main = paste0("Genes summary: ", cohorts.list[i]), Rowv=NULL, Colv=NULL, scale="none", dendrogram="none", trace="none", hide_colorbar = FALSE, fontsize_row = 8, label_names=c("Gene","Mutation_type","Count")) %>%
  layout(width  = 900, height = 600, margin = list(l=150, r=10, b=150, t=50, pad=4), titlefont = list(size=16), xaxis = list(tickfont=list(size=10)), yaxis = list(tickfont=list(size=10)))

  ##### Add plot to the list for htmlwidgets
  widges.list[[i]] <- as_widget(ggplotly(p))

     ##### Save the heatmap as html (PLOTLY)
  htmlwidgets::saveWidget(as_widget(p), paste0(outDir, "/MAF_gene_summary_heatmap_", cohorts.list[i], ".html"), selfcontained = TRUE)

  ##### Plotly option
  #p <- plot_ly(x = colnames(geneSummary), y = rownames(geneSummary), z = as.matrix(geneSummary), height = 600, type = "heatmap") %>%
  #layout(title = paste0("Genes summary: ", cohorts.list[i]), autosize = TRUE, margin = list(l=150, r=10, b=100, t=100, pad=4), showlegend = TRUE)

  #widges.list[[i]] <- ggplotly(p)
}
```

```r
##### Detach plotly package. Otherwise it clashes with other graphics devices
detach("package:heatmaply", unload=FALSE)
detach("package:plotly", unload=FALSE)

##### Print a list of htmlwidgets
widges.list
```

<br>

*Note, since html files are not supported by GitHub the follwoing plots are screenshots only*

![](summariseMAFs_files/figure-html/MAF_gene_summary_heatmap_TCGA-PAAD.png)<!-- -->

![](summariseMAFs_files/figure-html/MAF_gene_summary_heatmap_ICGC-PACA-AU.png)<!-- -->

![](summariseMAFs_files/figure-html/MAF_gene_summary_heatmap_ICGC-PACA-AU-additional.png)<!-- -->

![](summariseMAFs_files/figure-html/MAF_gene_summary_heatmap_ICGC-PACA-CA.png)<!-- -->

### Visualisation

This part creates a set of plots summarising individual MAF files.

#### MAF summary plot

A summary for MAF file displaying frequency of various mutation/SNV types/classes (top panel), the number of variants in each sample as a stacked bar-plot (bottom-left) and variant types as a box-plot (bottom-middle), as well as the frequency of different mutation types for the top 10 mutated genes (bottom-right). The horizontal dashed line in stacked bar-plot represents median number of variants across the cohort.


```r
###### Generate separate plot for each cohort
for ( i in 1:length(mafFiles) ) {

  cat(paste(cohorts.list[i], "cohort\n\n", sep=" "))
  ##### Plotting MAF summary
  par(mar=c(4,4,2,0.5), oma=c(1.5,2,2,1))
  maftools::plotmafSummary(maf = mafInfo[[i]], rmOutlier = TRUE, addStat = 'median', dashboard = TRUE, titvRaw = FALSE)
  mtext("MAF summary", outer=TRUE,  cex=1, line=-0.5)
}
```

```
## TCGA-PAAD cohort
```

![](summariseMAFs_files/figure-html/maf_summary_plot-1.png)<!-- -->

```
## ICGC-PACA-AU cohort
```

![](summariseMAFs_files/figure-html/maf_summary_plot-2.png)<!-- -->

```
## ICGC-PACA-AU-additional cohort
```

![](summariseMAFs_files/figure-html/maf_summary_plot-3.png)<!-- -->

```
## ICGC-PACA-CA cohort
```

![](summariseMAFs_files/figure-html/maf_summary_plot-4.png)<!-- -->

#### Oncoplot

Oncoplot illustrating different types of mutations observed across samples for the 10 most frequently mutated genes. The side and top bar-plots present the frequency of mutations in each gene and in each sample, respectively.


```r
###### Generate separate plot for each cohort
for ( i in 1:length(mafFiles) ) {

  cat(paste(cohorts.list[i], "cohort\n\n", sep=" "))

  ##### Drawing oncoplots for the top 10 genes in each cohort
  plot.new()
  par(mar=c(4,4,2,0.5), oma=c(1.5,2,2,1))
  maftools::oncoplot(maf = mafInfo[[i]], top = 10, fontSize = 12)
}
```

```
## TCGA-PAAD cohort
```

![](summariseMAFs_files/figure-html/maf_oncoplot-1.png)<!-- -->

```
## ICGC-PACA-AU cohort
```

![](summariseMAFs_files/figure-html/maf_oncoplot-2.png)<!-- -->

```
## ICGC-PACA-AU-additional cohort
```

![](summariseMAFs_files/figure-html/maf_oncoplot-3.png)<!-- -->

```
## ICGC-PACA-CA cohort
```

![](summariseMAFs_files/figure-html/maf_oncoplot-4.png)<!-- -->

#### Transition and transversions distribution plot

Plots presenting the transition and transversions distribution. The box-plots (top panel) show the overall distribution of the six different conversions (C>A, C>G, C>T, T>C, T>A and T>G)(left), and the transition and transversions frequency (right). The stacked bar-plot (bottom) displays the fraction of the six different conversions in each sample.


```r
###### Generate separate plot for each cohort
for ( i in 1:length(mafFiles) ) {

  cat(paste(cohorts.list[i], "cohort\n\n", sep=" "))

  ##### Drawing distribution plots of the transitions and transversions
  titv.info <- maftools::titv(maf = mafInfo[[i]], plot = FALSE, useSyn = TRUE)

  maftools::plotTiTv(res = titv.info)
  mtext("Transition and transversions distribution", outer=TRUE,  cex=1, line=-1.5)
}
```

```
## TCGA-PAAD cohort
```

![](summariseMAFs_files/figure-html/maf_TiTv_plot-1.png)<!-- -->

```
## ICGC-PACA-AU cohort
```

![](summariseMAFs_files/figure-html/maf_TiTv_plot-2.png)<!-- -->

```
## ICGC-PACA-AU-additional cohort
```

![](summariseMAFs_files/figure-html/maf_TiTv_plot-3.png)<!-- -->

```
## ICGC-PACA-CA cohort
```

![](summariseMAFs_files/figure-html/maf_TiTv_plot-4.png)<!-- -->

#### Comparison with TCGA cohorts

Plot illustrating the mutation load in ICGC PACA-CA cohort along distribution of variants compiled from over 10,000 WXS samples across 33 TCGA landmark cohorts. Every dot represents a sample whereas the red horizontal lines are the median numbers of mutations in the respective cancer types. The vertical axis (log scaled) shows the number of mutations per megabase whereas the different cancer types are ordered on the horizontal axis based on their median numbers of somatic mutations. This plot is similar to the one described in the paper "Signatures of mutational processes in human cancer" by Alexandrov et al. (PMID: 23945592)

```r
###### Generate separate plot for each cohort
for ( i in 1:length(mafFiles) ) {

  ##### Compare mutation load against TCGA cohorts
  maftools::tcgaCompare(maf = mafInfo[[i]], cohortName = cohorts.list[i], primarySite=TRUE)
}
```

```
##                      Cohort Cohort_Size Median_Mutations
##  1:                    Skin         468            315.0
##  2:           Lung Squamous         494            187.5
##  3:              Lung Adeno         567            158.0
## ...
## 12:                   Liver         374             67.0
## 13:                  Cervix         305             66.0
## 14:       Colorectal Rectum         158             63.0
## 15:         Kidney Papilary         288             53.0
## 16:               TCGA-PAAD         143             47.0
## 17:       Kidney Clear Cell         339             44.0
## 18:                  Uterus          57             35.0
## ...
## 34:           Adrenal Gland         177              7.0
##                      Cohort Cohort_Size Median_Mutations
```

![](summariseMAFs_files/figure-html/maf_tcga_cohorts-1.png)<!-- -->

```
##                      Cohort Cohort_Size Median_Mutations
##  1:                    Skin         468            315.0
##  2:           Lung Squamous         494            187.5
## ...
## 22:                  Pleura          83             25.0
## 23:                Pancreas         178             22.5
## 24: Adrenal Gland Carcinoma          92             21.5
## 25:            Brain Glioma         511             19.0
## 26:                Prostate         496             19.0
## 27:            ICGC-PACA-AU         395             18.0
## 28:      Kidney Chromophobe          66             13.0
## 29:                  Testis         149             11.0
## ...
## 34:           Adrenal Gland         177              7.0
##                      Cohort Cohort_Size Median_Mutations
```

![](summariseMAFs_files/figure-html/maf_tcga_cohorts-2.png)<!-- -->

```
##                      Cohort Cohort_Size Median_Mutations
##  1:                    Skin         468            315.0
##  2:           Lung Squamous         494            187.5
## ...
## 14:       Colorectal Rectum         158             63.0
## 15:         Kidney Papilary         288             53.0
## 16:       Kidney Clear Cell         339             44.0
## 17: ICGC-PACA-AU-additional          25             40.0
## 18:                  Uterus          57             35.0
## 19:                  Breast        1044             34.0
## 20:      Brain Glioblastoma         395             32.0
## ...
## 33:                 Thyroid         491              9.0
## 34:           Adrenal Gland         177              7.0
##                      Cohort Cohort_Size Median_Mutations
```

![](summariseMAFs_files/figure-html/maf_tcga_cohorts-3.png)<!-- -->

```
##                      Cohort Cohort_Size Median_Mutations
##  1:                    Skin         468            315.0
##  2:           Lung Squamous         494            187.5
## ...
## 16:       Kidney Clear Cell         339             44.0
## 17:                  Uterus          57             35.0
## 18:                  Breast        1044             34.0
## 19:      Brain Glioblastoma         395             32.0
## 20:             Soft Tissue         255             31.0
## 21:               Bile Duct          51             30.0
## 22:                  Pleura          83             25.0
## 23:            ICGC-PACA-CA         336             25.0
## 24:                Pancreas         178             22.5
## ...
## 34:           Adrenal Gland         177              7.0
##                      Cohort Cohort_Size Median_Mutations
```

![](summariseMAFs_files/figure-html/maf_tcga_cohorts-4.png)<!-- -->

<br>

Print session info


```r
sessionInfo()
```

```
## R version 3.5.0 (2018-04-23)
## Platform: x86_64-apple-darwin15.6.0 (64-bit)
## Running under: macOS High Sierra 10.13.4
## 
## Matrix products: default
## BLAS: /Library/Frameworks/R.framework/Versions/3.5/Resources/lib/libRblas.0.dylib
## LAPACK: /Library/Frameworks/R.framework/Versions/3.5/Resources/lib/libRlapack.dylib
## 
## locale:
## [1] en_AU.UTF-8/en_AU.UTF-8/en_AU.UTF-8/C/en_AU.UTF-8/en_AU.UTF-8
## 
## attached base packages:
## [1] parallel  stats     graphics  grDevices utils     datasets  methods  
## [8] base     
## 
## other attached packages:
## [1] knitr_1.20          optparse_1.4.4      xlsx_0.5.7         
## [4] xlsxjars_0.6.1      rJava_0.9-9         maftools_1.6.07    
## [7] Biobase_2.40.0      BiocGenerics_0.26.0
```