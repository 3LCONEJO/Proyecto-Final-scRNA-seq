# REPORTE DE ANÁLISIS COMPUTACIONAL AVANZADO
## Human striatal glia differentially contribute to AD- and PD-specific neurodegeneration
#### Emiliano Ferro, Sofia Gamiño, Irvin Rufino, Jorge Suazo

> Objetos Seurat en ken '/mnt/data/transcriptomica/eferro/SEURAT/CopyOfseurat_objects'

## Fase 0: Configuración del Entorno

Antes de iniciar el análisis transcriptómico, se establece un entorno computacional robusto mediante la creación de una estructura jerárquica de directorios de salida (QC_results/, seurat_objects/, markers_results/, clustering_results/, DEG_results/) y la carga de librerías especializadas. Este paso es crítico para mantener la trazabilidad del análisis y evitar sobrescrituras accidentales de datos intermedios. Se definen además paletas de colores globales tanto para condiciones patológicas (CTRL, AD, PD) como para tipos celulares que garantizan consistencia visual entre gráficos downstream.

## Fase 1: Adquisición y Organización de Datos Crudos

Los archivos raw de snRNA-seq se descargan desde repositorios públicos (GEO, SRA) en formato 10x Genomics estándar (matrices de conteos comprimidas: matrix.mtx.gz, barcodes.tsv.gz, features.tsv.gz). Estos archivos se reorganizan en subcarpetas por muestra biológica, se renombran conforme al estándar de la plataforma y se leen como matrices sparse. Posteriormente, se construye un objeto Seurat integrado mediante fusión de matrices individuales, asignando metadatos asociados (condición clínica, identificador de donante, etc.) a cada célula.

## Fase 2: Control de Calidad a Nivel de Célula

Se calcula un conjunto comprehensivo de métricas de control de calidad para cada núcleo aislado:

Complejidad génica (nFeature_RNA): Número de genes únicos detectados por célula, indicador de viabilidad y completitud transcriptómica.
Abundancia de transcritos (nCount_RNA): Número total de moléculas de ARN capturadas, proxy de la profundidad de secuenciación por célula.
Contaminación mitocondrial (percent_mito): Porcentaje de lecturas asignadas a genes mitocondriales, marcador de estrés celular o lisis citoplásmática.
Contaminación ribosomal (percent_ribo): Porcentaje de genes ribosomales, particularmente relevante en snRNA-seq donde ribosomas citoplásmicos indican contaminación.

Se establecen umbrales empíricos basados en el contexto biológico (tejido estriatal post-mortem) que discriminan células reales de artefactos técnicos sin sesgos selectivos de eliminación de poblaciones patológicas. Se visualizan distribuciones mediante gráficos de violín y scatter plots, facilitando la identificación de outliers y efectos de lote.

## Fase 3: Normalización y Transformación de la Expresión

Tras aplicar filtros de calidad, se normaliza la expresión génica mediante un marco robusto de regresión binomial negativa (SCTransform) que estabiliza la varianza mientras regresa variables confundentes (nCount_RNA, percent_mito). Este enfoque es superior a normalización logarítmica simple cuando existe sobredispersión masiva, como ocurre en cerebro post-mortem con heterogeneidad patológica. SCTransform genera una matriz de expresión "datos transformados" que preserva el ruido de genes débilmente expresados sin inflar artefactos técnicos.

## Fase 4: Selección de Genes y Reducción Dimensional (PCA)

Se identifican los genes más altamente variables (típicamente 3,000) entre muestras, priorizando característicos de cada tipo celular y condición patológica. Estos genes constituyen las "features de integración" y alimentan un Análisis de Componentes Principales (PCA) que comprime la matriz de expresión de miles de genes a decenas de componentes ortogonales, reteniendo ejes de máxima varianza. Se retienen 30 PCAs para preservar heterogeneidad intra-poblacional sutil (p.ej., subpoblaciones microgliales en transición).

## Fase 5: Integración de Datos entre Lotes (CCA + SCTransform)

Dado que las muestras provienen de donadores distintos y procesadas en lotes separados, se implementa integración por Canonical Correlation Analysis (CCA) adaptado a SCTransform. Este algoritmo:

Identifica "anclajes" (pares de células entre lotes con perfiles transcriptómicos similares)
Corrige el espacio de expresión para alinear estos anclajes
Preserva variación biológica legítima mientras mitiga sesgos técnicos

El parámetro k.filter se ajusta a 20 (vs default de 200) para mejorar precisión en identificación de tipos raros, crítico en contextos neuropatológicos.

## Fase 6: Reducción No Lineal y Visualización (UMAP)

Se aplica Uniform Manifold Approximation and Projection (UMAP) al espacio PCA integrado, comprimiendo la información de 30 dimensiones a 2 coordenadas Euclidianas que preservan topología local. UMAP revela agrupamientos celulares, intermezcla de condiciones clínicas y arquitectura de tipos celulares, permitiendo inspección visual de la calidad de integración.

## Fase 7: Clustering y Definición de Poblaciones

Se ejecuta el algoritmo de clustering basado en grafos de vecindad (FindNeighbors + FindClusters con resolución baja, típicamente 0.1) que particiona el espacio PCA en clusters de similitud alta. Resoluciones bajas generan clusters amplios que se refinan posteriormente mediante anotación por referencia, evitando sobreclustering impulsado por umbrales arbitrarios en lugar de biología.

## Fase 8: Anotación de Tipos Celulares

Se transfieren etiquetas de tipo celular desde referencias anotadas públicamente (p.ej., Azimuth con paneles cerebrales humanos) mediante predicción supervisada. Se asigna un tipo celular dominante por cluster basado en consenso Azimuth más abundante. Se eliminan clusters artefactados (etiquetados como "False" o "Splatter neuron") que representan agregados o dobletes no detectados previamente.

## Fase 9: Identificación de Marcadores por Cluster

Se ejecuta FindAllMarkers, comparando la expresión de cada cluster contra todos los demás, empleando test no paramétricos (Wilcoxon) que no asumen normalidad de distribuciones. Se retienen genes con log2FC > 0.2, expresados en ≥10% de células de su cluster e ≥20% más altos que en otros clusters. Se visualizan mediante heatmaps y dotplots que sintetizan identidades clusterales.

## Fase 10: Análisis de Expresión Diferencial (DEG) por Tipo Celular

Para cada tipo celular anotado, se ejecutan tres contrastes independientes: AD vs Control, PD vs Control, AD vs PD. Se utiliza test de Wilcoxon por rank (robusto a distribuciones asimétricas típicas en expresión génica), con corrección de múltiples testing (FDR < 0.05) y threshold empírico de log2FC ≥ 0.25. Se genera una tabla consolidada de DEGs que captura divergencias transcriptómicas patología-específicas.

## Fase 11: Enriquecimiento Funcional (GO, KEGG)

Los genes diferencialmente expresados se proyectan a ontologías funcionales (Gene Ontology para procesos biológicos, KEGG para vías metabólicas). Se calcula la probabilidad de que frecuencias observadas de genes bajo ciertos términos funcionales exceda lo esperado por azar, usando distribuciones hipergeométricas. Simplificación de términos GO redundantes mediante clustering semántico (cutoff Jaccard de 0.7) reduce ruido e identifica procesos biológicos centrales.

## Fase 12: Integración de Resultados y Generación de Reportes

Se sintetizan hallazgos dispersos en tablas y visualizaciones curadas (volcano plots, heatmaps, dotplots) que narran la historia de divergencias gliales entre AD y PD. Se guardan objetos Seurat integrados y anotados, matrices de expresión diferencial, y resultados de enriquecimiento en formatos tabulares (CSV) para interpretación posterior y validación experimental.

#### Human striatal glia differentially contribute to AD- and PD-specific neurodegeneration, por Jinbin Xu et. al.

Xu, J., Farsad, H. L., Hou, Y., Barclay, K., Lopez, B. A., Yamada, S., Saliu, I. O., Shi, Y., Knight, W. C., Bateman, R. J., Benzinger, T. L. S., Yi, J. J., Li, Q., Wang, T., Perlmutter, J. S., Morris, J. C., & Zhao, G. (2023). Human striatal glia differentially contribute to AD- and PD-specific neurodegeneration. Nature Aging, 3(3), 346–365. https://doi.org/10.1038/s43587-023-00363-8

```{r}
== QC_results/ ===
> list.files("QC_results/")
[1] "condition_umap.jpeg"      "ElbowPlot.jpeg"           "postQC_violins.jpeg"     
[4] "QC_FeatureScatter.jpeg"   "qc_metrics_filtered.csv"  "QC_violins.jpeg"         
[7] "top_genes_per_sample.csv" "umap_combined.jpeg"      
> 
> cat("\n=== seurat_objects/ ===\n")

=== seurat_objects/ ===
> list.files("seurat_objects/")
[1] "neuro_annotated.rds"      "neuro_integrated_cca.rds" "seurat_raw.rds"          
> 
> cat("\n=== clustering_results/ ===\n")

=== clustering_results/ ===
> list.files("clustering_results/")
[1] "Cell_Composition.jpeg"    "celltype_proportions.csv" "cluster_celltype_map.csv"
[4] "condition_umap.jpeg"      "FeaturePlot.jpeg"         "umap_azimuth.jpeg"       
[7] "umap_by_condition.jpeg"   "umap_coordinates.csv"    
> 
> cat("\n=== markers_results/ ===\n")

=== markers_results/ ===
> list.files("markers_results/")
[1] "all_markers_by_cluster.csv"  "CC_Violin.jpeg"              "Dotplot.jpeg"               
[4] "Heatmap.jpeg"                "top5_markers_by_cluster.csv"
> 
> cat("\n=== DEG_results/ ===\n")

=== DEG_results/ ===
> list.files("DEG_results/")
 [1] "ADPD_CTRLAstro_KEGG.jpeg"           "ADPD_CTRLAstro.jpeg"               
 [3] "de_all.rds"                         "DEG_barplot.jpeg"                  
 [5] "DEG_sig_AD_vs_Ctrl.csv"             "DEG_sig_AD_vs_PD.csv"              
 [7] "DEG_sig_PD_vs_Ctrl.csv"             "DEG_wilcox_all_celltypes.csv"      
 [9] "DEGVolc_Astrocyte.jpeg"             "DEGVolc_D1MSN.jpeg"                
[11] "DEGVolc_Microglia.jpeg"             "DEGVolc_Oligodendrocyte.jpeg"      
[13] "GO_compareCluster_Astrocytes.csv"   "GO_enrichment_results.csv"         
[15] "Heatmap.jpeg"                       "KEGG_compareCluster_Astrocytes.csv"
[17] "TopGO_DOT.jpeg"



R version 4.6.0 (2026-04-24)
Platform: x86_64-pc-linux-gnu
Running under: CachyOS

Matrix products: default
BLAS:   /usr/lib/libblas.so.3.12.0 
LAPACK: /usr/lib/liblapack.so.3.12.0  LAPACK version 3.12.0

locale:
 [1] LC_CTYPE=es_MX.UTF-8       LC_NUMERIC=C               LC_TIME=es_MX.UTF-8       
 [4] LC_COLLATE=es_MX.UTF-8     LC_MONETARY=es_MX.UTF-8    LC_MESSAGES=es_MX.UTF-8   
 [7] LC_PAPER=es_MX.UTF-8       LC_NAME=C                  LC_ADDRESS=C              
[10] LC_TELEPHONE=C             LC_MEASUREMENT=es_MX.UTF-8 LC_IDENTIFICATION=C       

time zone: America/Mexico_City
tzcode source: system (glibc)

attached base packages:
[1] stats4    grid      stats     graphics  grDevices utils     datasets  methods   base     

other attached packages:
 [1] AzimuthAPI_0.2.0       future_1.70.0          metap_1.14             zellkonverter_1.22.0  
 [5] DoubletFinder_2.0.6    ggh4x_0.3.1            enrichplot_1.32.0      org.Hs.eg.db_3.23.1   
 [9] AnnotationDbi_1.74.0   IRanges_2.46.0         S4Vectors_0.50.1       clusterProfiler_4.20.0
[13] circlize_0.4.18        ComplexHeatmap_2.28.0  scales_1.4.0           viridis_0.6.5         
[17] viridisLite_0.4.3      RColorBrewer_1.1-3     ggrepel_0.9.8          Seurat_5.5.0          
[21] SeuratObject_5.4.0     sp_2.2-1               patchwork_1.3.2        lubridate_1.9.5       
[25] forcats_1.0.1          stringr_1.6.0          purrr_1.2.2            readr_2.2.0           
[29] tidyr_1.3.2            tibble_3.3.1           ggplot2_4.0.3          tidyverse_2.0.0       
[33] dplyr_1.2.1            rhdf5_2.56.0           hdf5r_1.3.12           GEOquery_2.80.0       
[37] Biobase_2.72.0         BiocGenerics_0.58.1    generics_0.1.4         Matrix_1.7-5          

loaded via a namespace (and not attached):
  [1] R.methodsS3_1.8.2           goftest_1.2-3               DT_0.34.0                  
  [4] Biostrings_2.80.1           HDF5Array_1.40.0            TH.data_1.1-5              
  [7] vctrs_0.7.3                 ggtangle_0.1.2              spatstat.random_3.5-0      
 [10] digest_0.6.39               png_0.1-9                   shape_1.4.6.1              
 [13] gypsum_1.8.0                deldir_2.0-4                parallelly_1.47.0          
 [16] MASS_7.3-65                 fontLiberation_0.1.0        reshape2_1.4.5             
 [19] httpuv_1.6.17               foreach_1.5.2               qvalue_2.44.0              
 [22] withr_3.0.2                 ggrastr_1.0.2               xfun_0.58                  
 [25] ggfun_0.2.0                 survival_3.8-6              memoise_2.0.1              
 [28] ggbeeswarm_0.7.3            Seqinfo_1.2.0               gson_0.1.0                 
 [31] systemfonts_1.3.2           ragg_1.5.2                  tidytree_0.4.7             
 [34] zoo_1.8-15                  GlobalOptions_0.1.4         argparse_2.3.1             
 [37] pbapply_1.7-4               R.oo_1.27.1                 KEGGREST_1.52.0            
 [40] promises_1.5.0              otel_0.2.0                  httr_1.4.8                 
 [43] globals_0.19.1              fitdistrplus_1.2-6          rhdf5filters_1.24.0        
 [46] ps_1.9.3                    rstudioapi_0.18.0           miniUI_0.1.2               
 [49] DOSE_4.6.0                  dir.expiry_1.20.0           processx_3.9.0             
 [52] fields_17.3                 curl_7.1.0                  h5mread_1.4.0              
 [55] TFisher_0.2.0               polyclip_1.10-7             ExperimentHub_3.2.0        
 [58] SparseArray_1.12.2          xtable_1.8-8                doParallel_1.0.17          
 [61] evaluate_1.0.5              S4Arrays_1.12.0             BiocFileCache_3.2.0        
 [64] hms_1.1.4                   GenomicRanges_1.64.0        irlba_2.3.7                
 [67] colorspace_2.1-2            filelock_1.0.3              ROCR_1.0-12                
 [70] reticulate_1.46.0           spatstat.data_3.1-9         magrittr_2.0.5             
 [73] lmtest_0.9-40               glmGamPoi_1.24.0            later_1.4.8                
 [76] ggtree_4.2.0                lattice_0.22-9              spatstat.geom_3.8-1        
 [79] future.apply_1.20.2         scattermore_1.2             XML_3.99-0.23              
 [82] cowplot_1.2.0               matrixStats_1.5.0           RcppAnnoy_0.0.23           
 [85] pillar_1.11.1               nlme_3.1-169                iterators_1.0.14           
 [88] compiler_4.6.0              beachmat_2.28.0             RSpectra_0.16-2            
 [91] stringi_1.8.7               tensor_1.5.1                SummarizedExperiment_1.42.0
 [94] plyr_1.8.9                  crayon_1.5.3                abind_1.4-8                
 [97] gridGraphics_0.5-1          sn_2.1.3                    mathjaxr_2.0-0             
[100] bit_4.6.0                   sandwich_3.1-1              multcomp_1.4-30            
[103] textshaping_1.0.5           codetools_0.2-20            crosstalk_1.2.2            
[106] bslib_0.11.0                alabaster.ranges_1.12.0     GetoptLong_1.1.1           
[109] plotly_4.12.0               multtest_2.68.0             mime_0.13                  
[112] splines_4.6.0               Rcpp_1.1.1-1.1              fastDummies_1.7.6          
[115] basilisk_1.24.0             tidydr_0.0.6                dbplyr_2.5.2               
[118] sparseMatrixStats_1.24.0    utf8_1.2.6                  knitr_1.51                 
[121] blob_1.3.0                  clue_0.3-68                 BiocVersion_3.23.1         
[124] fs_2.1.0                    listenv_0.10.1              DelayedMatrixStats_1.34.0  
[127] Rdpack_2.6.6                ggplotify_0.1.3             callr_3.8.0                
[130] statmod_1.5.2               tzdb_0.5.0                  tweenr_2.0.3               
[133] pkgconfig_2.0.3             tools_4.6.0                 cachem_1.1.0               
[136] rbibutils_2.4.1             RSQLite_3.53.1              numDeriv_2016.8-1.1        
[139] DBI_1.3.0                   celldex_1.22.0              fastmap_1.2.0              
[142] rmarkdown_2.31              ica_1.0-3                   sass_0.4.10                
[145] AnnotationHub_4.2.0         BiocManager_1.30.27         dotCall64_1.2              
[148] RANN_2.6.2                  alabaster.schemas_1.12.0    SingleR_2.14.0             
[151] farver_2.1.2                scatterpie_0.2.6            yaml_2.3.12                
[154] MatrixGenerics_1.24.0       cli_3.6.6                   lifecycle_1.0.5            
[157] uwot_0.2.4                  mvtnorm_1.4-1               presto_1.0.0               
[160] timechange_0.4.0            gtable_0.3.6                rjson_0.2.23               
[163] ggridges_0.5.7              progressr_0.19.0            parallel_4.6.0             
[166] ape_5.8-1                   limma_3.68.4                enrichit_0.1.4             
[169] jsonlite_2.0.0              bitops_1.0-9                RcppHNSW_0.7.0             
[172] bit64_4.8.2                 qqconf_1.3.2                Rtsne_0.17                 
[175] yulab.utils_0.2.4           alabaster.matrix_1.12.0     spatstat.utils_3.2-3       
[178] aisdk_1.4.12                mutoss_0.1-14               jquerylib_0.1.4            
[181] alabaster.se_1.12.0         GOSemSim_2.38.0             spatstat.univar_3.2-0      
[184] R.utils_2.13.0              lazyeval_0.2.3              alabaster.base_1.12.0      
[187] shiny_1.13.0                htmltools_0.5.9             GO.db_3.23.1               
[190] sctransform_0.4.3           rappdirs_0.3.4              glue_1.8.1                 
[193] spam_2.11-4                 httr2_1.2.2                 XVector_0.52.0             
[196] RCurl_1.98-1.19             gdtools_0.5.1               treeio_1.36.1              
[199] mnormt_2.1.2                gridExtra_2.3               igraph_2.3.2               
[202] R6_2.6.1                    SingleCellExperiment_1.34.0 ggiraph_0.9.6              
[205] labeling_0.4.3              cluster_2.1.8.2             Rhdf5lib_2.0.0             
[208] aplot_0.2.9                 plotrix_3.8-14              DelayedArray_0.38.2        
[211] tidyselect_1.2.1            vipor_0.4.7                 maps_3.4.3                 
[214] ggforce_0.5.0               xml2_1.5.2                  fontBitstreamVera_0.1.1    
[217] KernSmooth_2.23-26          S7_0.2.2                    fontquiver_0.2.1           
[220] data.table_1.18.4           htmlwidgets_1.6.4           rlang_1.2.0                
[223] spatstat.sparse_3.2-0       spatstat.explore_3.8-1      rentrez_1.2.4              
[226] Cairo_1.7-0                 ggnewscale_0.5.2            beeswarm_0.4.0             
```
