An example how to build multiple super-cell-like objects, including
‘exact’ (Super-cells obtained with the exact coarse-gaining), ‘approx’
(Super-cells obtained with the aaproximate coarse-graining),
’metacell(.\*)’ (Metacell build on the same genes as super-cells –
‘metacell\_SC\_like’; and Metacell build in a default set of genes –
‘metacell\_default’) for a set of graining levels and random seeds.

``` r
# if (!requireNamespace("BiocManager", quietly = TRUE))
#     install.packages("BiocManager")
# 
# BiocManager::install("SingleCellExperiment")
# 
# if (!requireNamespace("remotes")) install.packages("remotes")
# remotes::install_github("GfellerLab/SuperCell")
# remotes::install_github("mariiabilous/SuperCellBM")

library(SingleCellExperiment)
library(SuperCell)
library(SuperCellBM)
```

Load some default parameters
----------------------------

Such as `.gamma.seq`for the set of fraining levels, `.seed.seq` for the
set of random seeds, `adata.folder` and `fig.folder` for the folders
where to write data and plots. For the full list of the default
parameters, see `./examples/config/Tian_config.R`.

``` r
source("./examples/config/Tian_config.R")
```

### Flags

Whether to compute super-cell (`ToComputeSC`) or whether to compute
super-cell gene expression (`ToComputeSC_GE`) or load saved files. Make
sure, these file exists :) Flag `ToTestPackage` is used to run stript in
2 modes: package testing (`ToTestPackage == TRUE`) or generating
super-cell structure and super-cell gene expression for the further
analyses (`ToTestPackage == FALSE`).

``` r
ToComputeSC <- T
ToComputeSC_GE <- T

ToTestPackage <- T # @Loc, This is just to test whether package works (on reduced set of graining levels and seeds, turn it to FALSE to get real data @ all grainig levels and seeds)

filename_suf <- "" # variable to add a suffix to the saved files in case of testing of the package

if(ToTestPackage){
  testing_gamma_seq <- c(1, 10, 100)
  testing_seed_seq <- .seed.seq[1:3]
  
  warning(paste("The reduced set of graining leveles and seeds will be used, to get real output, turn it ti FALSE"))
  warning(paste("Original set of graining levels is:", paste(.gamma.seq, collapse = ", "), 
                "but used testing set is:", paste(testing_gamma_seq, collapse = ", ")))
  warning(paste("Original set of seeds  is:", paste(.seed.seq, collapse = ", "), 
                "but used testing set is:", paste(testing_seed_seq, collapse = ", ")))
  
  .gamma.seq <- testing_gamma_seq
  .seed.seq <- testing_seed_seq
  
  filename_suf = "_testing_package"
}
```

    ## Warning: The reduced set of graining leveles and seeds will be used, to get real
    ## output, turn it ti FALSE

    ## Warning: Original set of graining levels is: 1, 2, 5, 10, 20, 50, 100, 200 but
    ## used testing set is: 1, 10, 100

    ## Warning: Original set of seeds is: 12345, 111, 19, 42, 7, 559241, 123, 987, 234,
    ## 91, 877, 451, 817 but used testing set is: 12345, 111, 19

Load `cell_lines` data from [Tian et al., 2019](https://pubmed.ncbi.nlm.nih.gov/31133762/).
-------------------------------------------------------------------------------------------

``` r
RData.file.path <- file.path(data.folder, 'cell_lines_git.RData')

if(!file.exists(RData.file.path)){
  if(!dir.exists(data.folder)) dir.create(data.folder, recursive = T)
  download.file('https://github.com/LuyiTian/sc_mixology/blob/master/data/sincell_with_class_5cl.RData?raw=true', 
                RData.file.path)
}

load(RData.file.path)

# keep used dataset 
cell_lines_SCE <- sce_sc_10x_5cl_qc

#remove not-used datasets
rm(sc_Celseq2_5cl_p1, sc_Celseq2_5cl_p2, sc_Celseq2_5cl_p3, sce_sc_10x_5cl_qc)
```

Get and set the main variables, such as single-cell gene expression
(`sc.GE`), single-cell counts (`sc.counts`), number of single cells
(`N.c`) and total number of genes (`N.g`). Set matrix column names to
cellIDs (`cell.ids`) and row names to gene names (`gene.names`).

``` r
cell.ids     <- cell_lines_SCE@colData@rownames
N.c          <- cell_lines_SCE@colData@nrows 


gene.names   <- cell_lines_SCE@int_elementMetadata$external_gene_name
N.g          <- length(gene.names)

sc.GE           <- cell_lines_SCE@assays$data$logcounts
colnames(sc.GE) <- cell.ids
rownames(sc.GE) <- gene.names

sc.counts   <- cell_lines_SCE@assays$data$counts
colnames(sc.counts) <- cell.ids
rownames(sc.counts) <- gene.names

## this is not needed at this point, but will be used later
GT.cell.type <- cell_lines_SCE@colData$cell_line_demuxlet
names(GT.cell.type) <- cell.ids
N.clusters         <- length(unique(GT.cell.type))

GT.cell.type.names          <- names(table(GT.cell.type))
GT.cell.type.2.num          <- 1:length(unique(GT.cell.type))
names(GT.cell.type.2.num)   <- GT.cell.type.names
GT.cell.type.num            <- GT.cell.type.2.num[GT.cell.type]
names(GT.cell.type.num)     <- names(GT.cell.type)

## uncomment this when needed 
#.pal.GT <- .color.tsne.Tian  ## to Global Config
#scales::show_col(.pal.GT)


#mito.genes <-  grep(pattern = "^MT", x = gene.names, value = TRUE)
#ribo.genes <-  grep(pattern = "^RP[LS]", x = gene.names, value = TRUE)
#mito.ribo.genes <- c(mito.genes, ribo.genes)
#length(mito.ribo.genes)

#gene.meta <- data.frame(name = gene.names, inNcells = rowSums(sc.GE>0), mean.expr = rowMeans(sc.GE), sd = rowSds(sc.GE))
#head(gene.meta)
```

Compute Super-cells structure
-----------------------------

for the Exact, Aprox (Super-cells obtained with the exact or approximate
coarse-graining), Subsampling or Random (random grouping of cells into
super-cells).

``` r
filename <- paste0('initial', filename_suf)

SC.list <- compute_supercells(
  sc.GE,
  ToComputeSC = ToComputeSC,
  data.folder = data.folder,
  filename = filename,
  gamma.seq = .gamma.seq,
  n.var.genes = .N.var.genes,
  k.knn = .k.knn,
  n.pc = .N.comp,
  approx.N = .approx.N,
  fast.pca = TRUE,
  genes.use = .genes.use, 
  genes.exclude = .genes.omit,
  seed.seq = .seed.seq
  )

cat(paste("Super-cell computed for:", paste(names(SC.list), collapse = ", "), 
          "\nat graining levels:", paste(names(SC.list[['Approx']]), collapse = ", "),
          "\nfor seeds:", paste(names(SC.list[['Approx']][[1]]), collapse = ", "), "\n",
          "\nand saved to / loaded from", paste0(filename, ".Rds")))
```

    ## Super-cell computed for: Exact, Approx, Random, Subsampling 
    ## at graining levels: 1, 10, 100 
    ## for seeds: 12345, 111, 19 
    ##  
    ## and saved to / loaded from initial_testing_package.Rds

Compute metacells in two settings:
----------------------------------

-   ‘metacell\_default’ - when metacell is computed with the default
    parameters (from the tutorial), using gene set filtered by MC
-   ‘metacell\_SC\_like’ - when metacell is computed with the same
    default parameters, but at the same set of genes as Super-cells

``` r
SC.mc <- compute_supercells_metacells(
  sc.counts = sc.counts, 
  gamma.seq = .gamma.seq,
  SC.list = SC.list,
  proj.name = proj.name,
  ToComputeSC = ToComputeSC, 
  mc.k.knn = 100,
  T_vm_def = 0.08,
  MC.folder = "MC", 
  MC_gene_settings = c('metacell_default', 'metacell_SC_like') # do not change
)
```

### Get actual graining levels obtained with Metacell

``` r
additional_gamma_seq <- get_actual_gammas_metacell(SC.mc)

cat(paste("Metacells were computed in", length(names(SC.mc)), "settings:", paste(names(SC.mc), collapse = ", "), 
          "\nfor Gammas:", paste(names(SC.mc[[1]]), collapse = ", "), 
          "\nbut actual gammas are:", paste(additional_gamma_seq, collapse = ", ")
))
```

    ## Metacells were computed in 2 settings: metacell_default, metacell_SC_like 
    ## for Gammas: 1, 10, 100 
    ## but actual gammas are: 46, 54, 69

``` r
# manually expand MC because later we will have 2 different setting for MC profile: fp - footpring of MC, av - averaged 
SC.mc.fp <- SC.mc
names(SC.mc.fp) <- sapply(names(SC.mc), FUN = function(x){paste0(x, '_fp')})

SC.mc.av <- SC.mc
names(SC.mc.av) <- sapply(names(SC.mc), FUN = function(x){paste0(x, '_av')})

SC.mc.expanded <- c(SC.mc.fp, SC.mc.av)

names(SC.mc.expanded)
```

    ## [1] "metacell_default_fp" "metacell_SC_like_fp" "metacell_default_av"
    ## [4] "metacell_SC_like_av"

``` r
rm(SC.mc.fp, SC.mc.av, SC.mc)
```

Compute super-cells (Exact, Approx, Subsampling and Random) at the addidional grainig levels obtained with Metacell.
--------------------------------------------------------------------------------------------------------------------

So that we can dirrectly compare the results of Super-cells and
Metacells at the same graining levels.

``` r
filename <- paste0('additional_gammas', filename_suf)

SC.list <- compute_supercells_additional_gammas(
  SC.list,
  additional_gamma_seq = additional_gamma_seq,
  ToComputeSC = ToComputeSC,
  data.folder = data.folder,
  filename = filename,
  approx.N = .approx.N,
  fast.pca = TRUE
)

cat(paste("Super-cells of methods:", paste(names(SC.list), collapse = ", "), 
      "\nwere computed at aggitional graining levels:", paste(additional_gamma_seq, collapse = ", "), 
      "\nand added to SC.list"
      ))
```

    ## Super-cells of methods: Exact, Approx, Random, Subsampling 
    ## were computed at aggitional graining levels: 46, 54, 69 
    ## and added to SC.list

### Concatenate Metacells to the list of Super-cells

``` r
SC.list <- c(SC.list, SC.mc.expanded)
rm(SC.mc.expanded)

filename <- paste0("all", filename_suf)
saveRDS(SC.list, file = file.path(data.folder, "SC", paste0(filename, ".Rds")))

cat(paste(
  "Metacell data added to SC.list \nand now it contains:",
  paste(names(SC.list), collapse = ", "),
  "\nSC.list was saved to", file.path(data.folder, "SC", paste0(filename, ".Rds"))
))
```

    ## Metacell data added to SC.list 
    ## and now it contains: Exact, Approx, Random, Subsampling, metacell_default_fp, metacell_SC_like_fp, metacell_default_av, metacell_SC_like_av 
    ## SC.list was saved to examples/data/Tian/SC/all_testing_package.Rds

Compute GE for Super-cell data
------------------------------

GE profile for the super-cell data is computede:

-   for super-cells (Exact, Approx) by averaging gene expression within
    super-cells,
-   for Random, also averaging gene expression within super-cells
-   for the Subsampling, sc.GE matrix is just subsampled,
-   for Metacells, gene expression is computed in 2 ways:
    1.  the same as super-cells (averaging gene expression within
        Metacells) -&gt; `metacell_(.*)_av`
    2.  using the default output of Metacell maned footprint -&gt;
        `metacell_(.*)_fp`

``` r
filename <- paste("all", filename_suf)

SC.GE.list <- compute_supercells_GE(
  sc.GE = sc.GE, 
  SC.list = SC.list,
  ToComputeSC_GE = ToComputeSC_GE, 
  data.folder = data.folder,
  filename = filename
)

cat(paste("Gene expression profile computed for:", paste(names(SC.GE.list), collapse = ", "), 
    "\nat graining levels:", paste(sort(as.numeric(names(SC.GE.list[['Approx']]))), collapse = ", "),
    "\nfor seeds:", paste(names(SC.GE.list[['Approx']][[1]]), collapse = ", "),
    "\nand saved to / loaded from", paste0(filename, ".Rds")
    ))
```

    ## Gene expression profile computed for: Exact, Approx, Random, Subsampling, metacell_default_fp, metacell_SC_like_fp, metacell_default_av, metacell_SC_like_av 
    ## at graining levels: 1, 10, 46, 54, 69, 100 
    ## for seeds: 12345, 111, 19 
    ## and saved to / loaded from all _testing_package.Rds

#### Final

    ## [1] "Done! Congrats!"

    ## Warning: (!) Script was run in a test mode, to get real cell_line super-cell
    ## data, run this script with ToTestPackage <- FALSE
