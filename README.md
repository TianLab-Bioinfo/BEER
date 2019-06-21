<img src="https://github.com/jumphone/BEER/raw/master/DATA/BEER_LOGO.png" width="200">

### BEER: Batch EffEct Remover for single-cell data

Author: Feng Zhang

BEER's latest version: https://github.com/jumphone/BEER/releases

BEER's manuscript version: https://github.com/jumphone/BEER/archive/0.0.2.zip

# News:

* June 2019 ( v0.0.8 ): BEER can be used to combine scATAC-seq & scRNA-seq

* ...

* June 2019 ( v0.0.4 ): "MBEER" is integrated into "BEER". Please directly use BEER to integrate multiple batches

* June 2019: stop updating the BEER source code for Seurat2. New feature is only for Seurat3.

# Requirement:

    #R >=3.5
    install.packages('Seurat') # >=3.0     
    install.packages('pcaPP') 
    install.packages('mclust')

# Usage:

* [I. Combine Two Batches](#I-Combine-Two-Batches)
* [II. Combine Multiple Batches](#II-Combine-Multiple-Batches)
* [III. UMAP-based Clustering](#III-UMAP-based-Clustering)
* [IV. Combine scATAC-seq & scRNA-seq](#iv-combine-scatac-seq--scrna-seq)
  
</br>

# I. Combine Two Batches

Download demo data: https://github.com/jumphone/BEER/raw/master/DATA/demodata.zip 

Please do basic quality control before using BEER (e.g. remove low-quality cells & genes). 

For QC, please see: https://satijalab.org/seurat/v3.0/pbmc3k_tutorial.html 

### Step1. Load Data

    library(Seurat)
  
    source('https://raw.githubusercontent.com/jumphone/BEER/master/BEER.R')
    
    #Load Demo Data (subset of GSE70630: MGH53 & MGH54)
    #Download: https://github.com/jumphone/BEER/raw/master/DATA/demodata.zip
    
    D1 <- read.table(unz("demodata.zip","DATA1_MAT.txt"), sep='\t', row.names=1, header=T)
    D2 <- read.table(unz("demodata.zip","DATA2_MAT.txt"), sep='\t', row.names=1, header=T)

    # "D1" & "D2" are UMI matrix (or FPKM, RPKM, TPM, PKM ...; Should not be gene-centric scaled data)
    # Rownames of "D1" & "D2" are gene names
    # Colnames of "D1" & "D2" are cell names 
    
    # There shouldn't be duplicated colnames in "D1" & "D2":
    colnames(D1)=paste0('D1_', colnames(D1))
    colnames(D2)=paste0('D2_', colnames(D2))

    DATA=.simple_combine(D1,D2)$combine
    BATCH=rep('D2',ncol(DATA))
    BATCH[c(1:ncol(D1))]='D1'

### Step2. Detect Batch Effect

    mybeer <- BEER(DATA, BATCH, GNUM=30, PCNUM=50, CPU=2)
    
    #GNUM: the number of groups in each batch. 
    #PCNUM: the number of computated PCA subspaces 
    
    #If you are combining data from different sequencing platforms or having "huge" batch effect, please try:
    #mybeer <- BEER(DATA, BATCH, GNUM=30, PCNUM=50, CPU=2, REGBATCH=TRUE)
    
    #Users can use "ReBEER" to re-run BEER faster with different parameters (REGBATCH and MTTAG can not be changed):
    #mybeer <- ReBEER(mybeer, GNUM=50, PCNUM=50, CPU=2)
    
    # Check selected PCs
    PCUSE=mybeer$select
    COL=rep('black',length(mybeer$cor))
    COL[PCUSE]='red'
    plot(mybeer$cor,mybeer$lcor,pch=16,col=COL,xlab='Rank Correlation',ylab='Linear Correlation',xlim=c(0,1),ylim=c(0,1))
    
    # Users can select PCA subspaces based on the distribution of "Rank Correlation" and "Linear Correlation". 
    # PCUSE=.getUSE(mybeer, CUTR=0.7, CUTL=0.7)
    
    
<img src="https://github.com/jumphone/BEER/raw/master/DATA/PLOT1.png" width="400">
    
### Step3. Visualization 
    
#### Keep batch effect:
  
<img src="https://github.com/jumphone/BEER/raw/master/DATA/PLOT2.png" width="400">
    
    pbmc <- mybeer$seurat
    ALLPC <- 1:length(mybeer$cor)   
    pbmc <- RunUMAP(object = pbmc, reduction.use='pca',dims = ALLPC, check_duplicates=FALSE)
    
    DimPlot(pbmc, reduction.use='umap', group.by='batch', pt.size=0.1)    
    #DimPlot(pbmc, reduction.use='umap', group.by='map', pt.size=0.1)
    


#### Remove batch effect:

<img src="https://github.com/jumphone/BEER/raw/master/DATA/PLOT3.png" width="400">

    pbmc <- mybeer$seurat
    PCUSE <- mybeer$select
    pbmc <- RunUMAP(object = pbmc, reduction.use='pca',dims = PCUSE, check_duplicates=FALSE)
    
    DimPlot(pbmc, reduction.use='umap', group.by='batch', pt.size=0.1)    
    #DimPlot(pbmc, reduction.use='umap', group.by='map', pt.size=0.1)
    
    
    
</br>
</br>
    
# II. Combine Multiple Batches

When combining multiple batches, BEER implements the iteration of "Combine Two Batches".

BEER compares each batch with the batch having the largest cell number.

The assumption is that the batch having the largest cell number has almost all cell-types within all batches.

If you want to define a batch having almost all cell-types, please set "MAXBATCH" to the label of that batch.

Download demo data: https://sourceforge.net/projects/beergithub/files/
   
### Step1. Load Data
    
    source('https://raw.githubusercontent.com/jumphone/BEER/master/BEER.R')
    
    #Load Demo Data (Oligodendroglioma, GSE70630)
    #Download: https://sourceforge.net/projects/beergithub/files/
    
    D1=readRDS('MGH36.RDS')
    D2=readRDS('MGH53.RDS')
    D3=readRDS('MGH54.RDS')
    D4=readRDS('MGH60.RDS')
    D5=readRDS('MGH93.RDS')
    D6=readRDS('MGH97.RDS')
    
    BATCH=c(rep('D1',ncol(D1)),
            rep('D2',ncol(D2)),
            rep('D3',ncol(D3)),
            rep('D4',ncol(D4)),
            rep('D5',ncol(D5)),
            rep('D6',ncol(D6)) )
            
    D12=.simple_combine(D1,D2)$combine
    D34=.simple_combine(D3,D4)$combine
    D56=.simple_combine(D5,D6)$combine
    D1234=.simple_combine(D12,D34)$combine
    D123456=.simple_combine(D1234,D56)$combine
    
    DATA=D123456   
   
    rm(D1);rm(D2);rm(D3);rm(D4);rm(D5);rm(D6)
    rm(D12);rm(D34);rm(D56);rm(D1234);rm(D123456)
    
### Step2. Detect Batch Effect

    mybeer=BEER(DATA, BATCH, MAXBATCH="", GNUM=30, PCNUM=50, CPU=2, SEED=1 )

    # Check selected PCs
    PCUSE=mybeer$select
    COL=rep('black',length(mybeer$cor))
    COL[PCUSE]='red'
    plot(mybeer$cor,mybeer$lcor,pch=16,col=COL,xlab='Rank Correlation',ylab='Linear Correlation',xlim=c(0,1),ylim=c(0,1))

<img src="https://github.com/jumphone/BEER/raw/master/DATA/PLOT4.png" width="400">

    
### Step3. Visualization 
        
#### Keep batch effect:
  
<img src="https://github.com/jumphone/BEER/raw/master/DATA/PLOT5.png" width="400">
    
    pbmc <- mybeer$seurat
    ALLPC <- 1:length(mybeer$cor)   
    pbmc <- RunUMAP(object = pbmc, reduction.use='pca',dims = ALLPC, check_duplicates=FALSE)
    
    DimPlot(pbmc, reduction.use='umap', group.by='batch', pt.size=0.1)    
    #DimPlot(pbmc, reduction.use='umap', group.by='map', pt.size=0.1)
    


#### Remove batch effect:

<img src="https://github.com/jumphone/BEER/raw/master/DATA/PLOT6.png" width="400">

    pbmc <- mybeer$seurat
    PCUSE <- mybeer$select
    pbmc <- RunUMAP(object = pbmc, reduction.use='pca',dims = PCUSE, check_duplicates=FALSE)
    
    DimPlot(pbmc, reduction.use='umap', group.by='batch', pt.size=0.1)    
    #DimPlot(pbmc, reduction.use='umap', group.by='map', pt.size=0.1)
    
    
</br>   
</br>


# III. UMAP-based Clustering
   
<img src="https://github.com/jumphone/BEER/raw/master/DATA/CLUST1.png" width="400">    

    VEC=pbmc@$reductions$umap@cell.embeddings
    
    # Here, we use the "dbscan" function to do clustering.
    library("fpc")
    set.seed(123)
    df=VEC
    db <- fpc::dbscan(df, eps = 0.5, MinPts = 5)
    DC=db$cluster
    pbmc@meta.data$DC=DC
    DimPlot(pbmc, reduction.use='umap', group.by='DC', pt.size=0.5)
    
</br>   
</br>

# IV. Combine scATAC-seq & scRNA-seq

Please go to the website of Seurat to download DEMO data: https://satijalab.org/seurat/v3.0/atacseq_integration_vignette.html
 
### Step1. Load Data

    source('https://raw.githubusercontent.com/jumphone/BEER/master/BEER.R')
    
    library(Seurat)
    library(ggplot2)
    peaks <- Read10X_h5("../data/atac_v1_pbmc_10k_filtered_peak_bc_matrix.h5")

    activity.matrix <- CreateGeneActivityMatrix(peak.matrix = peaks, 
        annotation.file = "../data/Homo_sapiens.GRCh37.82.gtf", 
        seq.levels = c(1:22, "X", "Y"), upstream = 2000, verbose = TRUE)
         
    pbmc.rna <- readRDS("../data/pbmc_10k_v3.rds")
    
    D1=as.matrix(activity.matrix)
    D2=as.matrix(pbmc.rna@assays$RNA@counts)
    colnames(D1)=paste0('ATAC_', colnames(D1))
    colnames(D2)=paste0('RNA_', colnames(D2))
    DATA=.simple_combine(D1,D2)$combine
    BATCH=rep('RNA',ncol(DATA))
    BATCH[c(1:ncol(D1))]='ATAC'
 
 
### Step2. Detect Batch Effect

    mybeer <- BEER(DATA, BATCH, GNUM=100, PCNUM=100, GN=5000, CPU=2, REGBATCH=TRUE)
    
    # Users can use "ReBEER" to adjust GNUM & PCNUM.
    # mybeer <- ReBEER(mybeer, GNUM=100, PCNUM=100, CPU=2)
    
    PCUSE=mybeer$select
    COL=rep('black',length(mybeer$cor))
    COL[PCUSE]='red'
    plot(mybeer$cor,mybeer$lcor,pch=16,col=COL,xlab='Rank Correlation',ylab='Linear Correlation',xlim=c(0,1),ylim=c(0,1))

<img src="https://github.com/jumphone/BEER/raw/master/DATA/PLOT7.png" width="400">   


### Step3. Visualization 
    
#### Keep batch effect:
  
<img src="https://github.com/jumphone/BEER/raw/master/DATA/PLOT8.png" width="400">
    
    pbmc <- mybeer$seurat
    DimPlot(pbmc, reduction.use='umap', group.by='batch', pt.size=0.1)    
    #DimPlot(pbmc, reduction.use='umap', group.by='map', pt.size=0.1)    

#### Remove batch effect:

<img src="https://github.com/jumphone/BEER/raw/master/DATA/PLOT9.png" width="400">

    pbmc <- mybeer$seurat
    #PCUSE <- .selectUSE(mybeer, CC=0.05)    
    PCUSE=mybeer$select
    pbmc <- RunUMAP(object = pbmc, reduction.use='pca',dims = PCUSE, check_duplicates=FALSE)
    
    DimPlot(pbmc, reduction.use='umap', group.by='batch', pt.size=0.1)    
    #DimPlot(pbmc, reduction.use='umap', group.by='map', pt.size=0.1)
    
    pbmc@meta.data$celltype=rep(NA,length(pbmc@meta.data$batch))
    pbmc@meta.data$celltype[which(pbmc@meta.data$batch=='RNA')]=pbmc.rna@meta.data$celltype
    
    DimPlot(pbmc, reduction.use='umap', group.by='celltype', pt.size=0.1,label=T)
    
<img src="https://github.com/jumphone/BEER/raw/master/DATA/PLOT10.png" width="400">


### Step4. Tune-up

If you need a "tune-up", please try BBKNN.

Details about BBKNN are in: https://github.com/Teichlab/bbknn

### Use BBKNN without BEER:

    umap=BEER.bbknn(mybeer, c(1:ncol(pbmc@reductions$pca@cell.embeddings)), NB=3, NT=10)
    pbmc@reductions$umap@cell.embeddings=umap
    DimPlot(pbmc, reduction.use='umap', group.by='celltype', pt.size=0.1,label=T)
    
<img src="https://github.com/jumphone/BEER/raw/master/DATA/PLOT12.png" width="400"> 


### Use BBKNN with BEER:
  
    umap=BEER.bbknn(mybeer, PCUSE, NB=3, NT=10)
    pbmc@reductions$umap@cell.embeddings=umap
    DimPlot(pbmc, reduction.use='umap', group.by='celltype', pt.size=0.1,label=T)

<img src="https://github.com/jumphone/BEER/raw/master/DATA/PLOT11.png" width="400"> 





</br>   
</br> 
    
    
# License
    
    MIT License
    
    Copyright (c) 2019 Zhang, Feng

    Permission is hereby granted, free of charge, to any person obtaining a copy
    of this software and associated documentation files (the "Software"), to deal
    in the Software without restriction, including without limitation the rights
    to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
    copies of the Software, and to permit persons to whom the Software is
    furnished to do so, subject to the following conditions:

    The above copyright notice and this permission notice shall be included in all
    copies or substantial portions of the Software.

    THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
    IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
    FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
    AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
    LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
    OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
    SOFTWARE.

  
