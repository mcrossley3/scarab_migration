Scarab ddRADseq data analysis
================

## Demultiplexing and basic quality control

We’ve just recieved A LOT (9.39 GB) of data: 100 bp single-end read
sequencing data from 239 samples of 11 species of dung beetle
(Scarabaeinae), multiplexed on a single Illumina lane, to be precise.
Before we dig in to the all the interesting questions we want to ask
with it, we should do a basic check that the sequencing reaction was
successful, and that we don’t need to start from scratch (lolsob).
First, we need to demultiplex the original file,
`raw_data/UTN-DUNG_S167_L007_R1_001.fastq.gz1`, using the list of
barcodes we appended to each sample during library preparation
(`ipyrad/barcodes`). We can look at it and see it matches our
expectations for .fastq data, which I won’t display here. (Enter all
shell commands in the appropriate directory via terminal.)

``` bash
gunzip -c ~/Dropbox/scarab_migration/raw_data/UTN-DUNG_S167_L007_R1_001.fastq.gz | head -n 12
```

We’ll initially use the program `ipyrad` [which has extensive
documentation you can view here](https://ipyrad.readthedocs.io/) to
demultiplex this file into our original libraries. We first generate a
params file.

``` bash
ipyrad -n scarab
```

We modify the file `ipyrad/params-scarab.txt` to match our data and fill
in the appropriate paths to our data and barcode file. We then run the
first step of the ipyrad only, which we indicate with the `-s` option:

``` bash
ipyrad -p params-scarab.txt -s 1
```

When this completes, you’ll see the relevant directory fill with fastq
files, and a text file labeled `s1_demultiplex_stats.txt` generated. We
want to extract data on the total number of reads per library, and group
libraries by species. First, we’ll read in the (unformatted) stats file
and select only the information we want:

``` r
file <- "~/Dropbox/scarab_migration/ipyrad/s1_demultiplex_stats.txt"
nsamples <- 239
raw <- readLines("~/Dropbox/scarab_migration/ipyrad/s1_demultiplex_stats.txt",warn=FALSE)
a <- grep("sample_name",raw) # where the data begins
a <- a[1] 
reads <- read.table(file,skip=(a),nrow=(nsamples))
colnames(reads) <- c("sample_ID", "read_count")
```

We know have a simple two column data frame to play around with. We will
do some tedious substitution of our field-based morphospecies assignment
shorthand with the real species IDs:

``` r
reads$sample_ID <- gsub("EU","Eurysternus_affin",reads$sample_ID)
reads$sample_ID <- gsub("DI1","Dichotomious_satanas",reads$sample_ID)
reads$sample_ID <- gsub("DI5","Dichotomious_satanas",reads$sample_ID)
reads$sample_ID <- gsub("DI7","Dichotomious_podalirius",reads$sample_ID)
reads$sample_ID <- gsub("DE2","Deltochilum_tesselatum",reads$sample_ID)
reads$sample_ID <- gsub("DE1","Deltochilum_speciosissimum",reads$sample_ID)
reads$sample_ID <- gsub("DE9","Deltochilum_speciosissimum",reads$sample_ID)
reads$sample_ID <- gsub("239","Dichotomious_protectus",reads$sample_ID)
reads$sample_ID <- gsub("238","Dichotomious_protectus",reads$sample_ID)
reads$sample_ID <- gsub("219","Deltochilum_amazonicum",reads$sample_ID)
reads$sample_ID <- gsub("220","Coprophanaeus_telamon",reads$sample_ID)
reads$sample_ID <- gsub("217","Phanaeus_meleagris",reads$sample_ID)
reads$sample_ID <- gsub("218","Phanaeus_meleagris",reads$sample_ID)
reads$sample_ID <- gsub("223","Oxysternon_silenus",reads$sample_ID)
reads$sample_ID <- gsub("222","Oxysternon_silenus",reads$sample_ID)
reads$sample_ID <- gsub("221","Oxysternon_conspicullatum",reads$sample_ID)
reads$sample_ID <- gsub('-.*', '', reads$sample_ID) # drop numbers
colnames(reads) <- c("Species","Read_count")
reads$Read_count <- as.numeric(as.character(reads$Read_count))
```

Kind of a mess, but that’s biology\! Now we’ll plot it.

![](scarab_analysis_notebook_files/figure-gfm/unnamed-chunk-6-1.png)<!-- -->

So far so good: pretty even sequencing effort across the five species
we’re interested in with the most data.

## Data assembly

Now that we’re comfortable we sequenced *something*, we’re going to try
and assemble the remainder of our data. We’re only going to analyze the
five species with reasonable sampling: *Deltochilum speciocissimum*,
*Deltochilum tesselatum*, *Dichotomius podalirius*, *Dichotomius
satanas*, and *Eurysternus affin*. We’ll run the rest of the assembly
and variant calling steps using [Jon Puritz’ dDocent
pipeline](https://www.ddocent.com/), which is optimized for nonmodel
organisms and does a good job maximizing the signal to noise ratio in
tricky denovo assemblies. First, we’ll make separate directories and
move our files (named by their initial morphospecies ID codes) to them.
We’ll then use a bash oneliner paired with files linking our current
sample names to a modified sample name format dDocent requires to get
ready to run the pipeline. Finally, we’ll run dDocent using a set of
config files for each species; these are largely the same, except for a
more lenient clustering threshhold for *D. satanas* based on exploratory
data analysis.

``` bash

# make directories
mkdir d_speciocissimum
mkdir d_tesselatum
mkdir d_podalirirus
mkdir d_satanas
mkdir e_affin

# move files
mv DE1* d_speciocissimum/
mv DE9* d_speciocissimum/ # two morphospecies lumped into one species
mv DE2* d_tesselatum/
mv DI7* d_podalirius/ 
mv DI1* d_satanas/
mv DI5* d_satanas/
mv EU* e_affin/ 

# rename files
cd d_speciocissimum/; eval "$(sed 's/^/mv /g' d_spec_rename.txt )"; cd ../.
cd d_tesselatum/; eval "$(sed 's/^/mv /g' d_tess_rename.txt )"; cd ../.
cd d_podalirius/; eval "$(sed 's/^/mv /g' d_pod_rename.txt )"; cd ../.
cd d_satanas/; eval "$(sed 's/^/mv /g' d_satanas_rename.txt )"; cd ../.
cd e_affin/; eval "$(sed 's/^/mv /g' e_affin_rename.txt )"; cd ../.

# run independently;
cd d_speciocissimum/; dDocent d_spec_config.file; cd ../.
cd d_tesselatum/; dDocent d_tess_config.file; cd ../.
cd d_podalirius/; dDocent d_pod_config.file; cd ../.
cd d_satanas/; dDocent d_satanas_config.file; cd ../.
cd e_affin/; dDocent e_affin_config.file; cd ../.
```

The output we’re interested in are .vcf files; we’ll be working with
these for the rest of the analysis. First, though, we’ll run a set of
standard filtering steps on the .vcf files for each species. We’re going
to drop any individuals with missing data at more than 30% of loci, any
loci missing data at more than 25% of individuals, all SNPs with a
minimum minor allele frequency of 0.05 and a minimum minor allele count
of 3, all SNPs with a quality (Phred) score of \<30, and any genotypes
with fewer than 5 reads across all individuals. We’ll then run dDocent’s
`dDocent_filter` command, which further trims for allelic balance,
depth, and other parameters you can read about in the [program’s
documentation](https://www.ddocent.com/filtering/). All are very
sensible. I’ve put all my commands together in a simple bash script,
`snp_filtering.sh`, but let the buyer beware: it’s written to be a
record of what I did, and not reproducible, although it would be easy
enough to tweak. For now, we’ll run it as I did, with a single command:

``` bash
bash scripts/snp_filtering.sh
```

Download these from your cluster, or whereever you’re running your
analyses. From here on out, we’ll be working in `R`.

## Plotting sampling localities

Before we start analyzing our genotypic data in R, it will be useful to
visualize where we collected our samples. We’ll do this using the
`ggmap` package and our coordinate data in `data/scarab_spp_master.csv`.
We’ll also set up a common color palette to use for the remainder of our
study. First, we’ll load our data, and use a Google Maps API to grab
contour maps of our field sites.

``` r
library(ggmap)
library(wesanderson)
library(gridExtra)

# read in locality data
localities <- read.csv("/Users/ethanlinck/Dropbox/scarab_migration/data/scarab_spp_master.csv")
levels(localities$locality) # see levels 
```

    ##  [1] "CC1-730"    "CC1-800"    "CC2-925"    "CC3-1050"   "CC4-1175"  
    ##  [6] "HU-1575"    "HU-1700"    "HU-1825"    "HU-1950"    "HU-3"      
    ## [11] "HU-4"       "Macucaloma" "Yanayacu"

``` r
localities$transect <- ifelse(grepl("CC",localities$locality),'colonso','pipeline') # add levels for transects
localities$population <-  as.factor(gsub( "_.*$", "", localities$ddocent_ID)) # add pops

# pick the center point for our maps, and an appropriate zoom level
pipeline <- get_map(location=c(lon=-77.85, lat=-0.615), zoom = 13, color = "bw")
colonso <- get_map(location=c(lon=-77.89, lat=-0.937), zoom = 14, color = "bw")

# subset locality data by transect
df.pipeline <- localities[localities$transect=='pipeline',]
df.colonso <- localities[localities$transect=='colonso',]
```

We’ll use the “BottleRocket2” palette from the [wesanderson
package](https://github.com/karthik/wesanderson), but build a custom
palette so we can hold colors constant across the levels of our sampling
localities.

``` r
# set palette
scale_fill_wes <- function(...){
  ggplot2:::manual_scale(
    'fill', 
    values = setNames(wes_palette("BottleRocket2", 13, type = "continuous"), levels(localities$short_locality)), 
    ...
  )
}

scale_color_wes <- function(...){
  ggplot2:::manual_scale(
    'color', 
    values = setNames(wes_palette("BottleRocket2", 13, type = "continuous"), levels(localities$short_locality)), 
    ...
  )
}
```

Now, we’ll plot these using [ggmap](https://github.com/dkahle/ggmap).

![](scarab_analysis_notebook_files/figure-gfm/unnamed-chunk-11-1.png)<!-- -->

The scaling is weird because of the constraints of the document but
you’ll have to trust me it’s looking good. Or at least about as good
as we’re going to get given where we’re working\!

## Testing for genetic differentiation

Now that we’re somewhat spatially oriented, we’ll dive in to analyzing
our genotypic data. Our driving question is whether dispersal (and gene
flow) is reduced across elevational gradients relative to within an
elevational band. We are going to try and answer this question in a
number of ways. An important first step is running principle component
analysis to ask whether there is population genetic structure, and
whether it relates to elevation. We’ll read in locality data, our .vcf
files, and run principle component analysis on each species in turn.
This will involve redundant chunks of code. First, *Dichotomius
satanas*, our species with the most throrough sampling (*n*=100).

``` r
# load libraries
library(vcfR)
library(adegenet)
library(ggplot2)
library(wesanderson)
library(tidyverse)
library(reshape2)
library(patchwork)

# set wd and read in locality data
localities <- read.csv("/Users/ethanlinck/Dropbox/scarab_migration/data/scarab_spp_master.csv")

### d. satanas ###

# read in .vcf and sample / pop names; convert to genlight
satanas.vcf <- read.vcfR("/Users/ethanlinck/Dropbox/scarab_migration/raw_data/d_satanas_filtered.FIL.recode.vcf", verbose = FALSE)

# pca 
satanas.dna <- vcfR2DNAbin(satanas.vcf, unphased_as_NA = F, consensus = T, extract.haps = F)
satanas.gen <- DNAbin2genind(satanas.dna)
satanas.pops <-  gsub( "_.*$", "", rownames(satanas.gen@tab))
satanas.gen@pop <- as.factor(satanas.pops)
satanas.scaled <- scaleGen(satanas.gen,NA.method="zero",scale=F)
satanas.pca <- prcomp(satanas.scaled,center=F,scale=F)
satanas.pc <- data.frame(satanas.pca$x[,1:3])
satanas.pc$sample <- rownames(satanas.pc)
satanas.pc$pop <- satanas.pops
satanas.pc$species <- rep("Dichotomius_satanas",nrow(satanas.pc))
```

We’ve run PCA and merged the first three PC axes with our sampling data.
Let’s plot PC1 and PC2, and also plot PC1 against elevation. (Note that
the colors will jump around during these initial plots, but will match
our sampling locality map once we’ve merged the different data frames.
So fear not\!)

![](scarab_analysis_notebook_files/figure-gfm/unnamed-chunk-13-1.png)<!-- -->

Looks like panmixia to me, but otherwise reasonable, with no wild
outliers. We’ll continue as before for the remaining four species. Next
up is *Deltochilum speciocissimum*.

``` r
# read in .vcf and sample / pop names; convert to genlight
spec.vcf <- read.vcfR("/Users/ethanlinck/Dropbox/scarab_migration/raw_data/d_spec_filtered.FIL.recode.vcf", verbose = FALSE)

# pca 
spec.dna <- vcfR2DNAbin(spec.vcf, unphased_as_NA = F, consensus = T, extract.haps = F)
spec.gen <- DNAbin2genind(spec.dna)
spec.pops <-  gsub( "_.*$", "", rownames(spec.gen@tab))
spec.gen@pop <- as.factor(spec.pops)
spec.scaled <- scaleGen(spec.gen,NA.method="mean",scale=F)
spec.pca <- prcomp(spec.scaled,center=F,scale=F)
spec.pc <- data.frame(spec.pca$x[,1:3])
spec.pc$sample <- rownames(spec.pc)
spec.pc$pop <- spec.pops
spec.pc$species <- rep("Deltochilum_speciocissimum",nrow(spec.pc))

# merge with sample data
spec.pc <- merge(spec.pc, localities, by.x = "sample", by.y = "ddocent_ID")
```

Plots:

![](scarab_analysis_notebook_files/figure-gfm/unnamed-chunk-15-1.png)<!-- -->

Looks reasonable. This time, we see strong popualtion genetic structure
that largely segregates across the Cosanga river. A curious exception is
a sample at the 2150 locality, which might represent a recent migrant.
Now, we’ll look at *Deltochilum tesselatum*.

``` r
# read in .vcf and sample / pop names; convert to genlight
tess.vcf <- read.vcfR("/Users/ethanlinck/Dropbox/scarab_migration/raw_data/d_tess_filtered.FIL.recode.vcf", verbose = FALSE)

# pca 
tess.dna <- vcfR2DNAbin(tess.vcf, unphased_as_NA = F, consensus = T, extract.haps = F)
tess.gen <- DNAbin2genind(tess.dna)
tess.pops <-  gsub( "_.*$", "", rownames(tess.gen@tab))
tess.gen@pop <- as.factor(tess.pops)
tess.scaled <- scaleGen(tess.gen,NA.method="mean",scale=F)
tess.pca <- prcomp(tess.scaled,center=F,scale=F)
tess.pc <- data.frame(tess.pca$x[,1:3])
tess.pc$sample <- rownames(tess.pc)
tess.pc$pop <- tess.pops
tess.pc$species <- rep("Deltochilum_tesselatum",nrow(tess.pc))

# merge with sample data
tess.pc <- merge(tess.pc, localities, by.x = "sample", by.y = "ddocent_ID")
```

Plots:

![](scarab_analysis_notebook_files/figure-gfm/unnamed-chunk-17-1.png)<!-- -->

Panmixia again, although there are fewer samples. Now, *Dichotomius
podalirius*.

``` r
# read in .vcf and sample / pop names; convert to genlight
pod.vcf <- read.vcfR("/Users/ethanlinck/Dropbox/scarab_migration/raw_data/d_pod_filtered.FIL.recode.vcf", verbose = FALSE)

# pca 
pod.dna <- vcfR2DNAbin(pod.vcf, unphased_as_NA = F, consensus = T, extract.haps = F)
pod.gen <- DNAbin2genind(pod.dna)
pod.pops <-  gsub( "_.*$", "", rownames(pod.gen@tab))
pod.gen@pop <- as.factor(pod.pops)
pod.scaled <- scaleGen(pod.gen,NA.method="mean",scale=F)
pod.pca <- prcomp(pod.scaled,center=F,scale=F)
pod.pc <- data.frame(pod.pca$x[,1:3])
pod.pc$sample <- rownames(pod.pc)
pod.pc$pop <- pod.pops
pod.pc$species <- rep("Dichotomius_podalirius",nrow(pod.pc))

# merge with sample data
pod.pc <- merge(pod.pc, localities, by.x = "sample", by.y = "ddocent_ID")
```

Plots:

![](scarab_analysis_notebook_files/figure-gfm/unnamed-chunk-19-1.png)<!-- -->

Again, a big mess. Note that we have fewer sampling localities here, and
that they are from the Colonso de Chalupas gradient. We’ll now run our
final species, *Eurysternus affin*, which is also from Colonso de
Chalupas.

``` r
# read in .vcf and sample / pop names; convert to genlight
affin.vcf <- read.vcfR("/Users/ethanlinck/Dropbox/scarab_migration/raw_data/e_affin_filtered.FIL.recode.vcf", verbose = FALSE)

# pca 
affin.dna <- vcfR2DNAbin(affin.vcf, unphased_as_NA = F, consensus = T, extract.haps = F)
affin.gen <- DNAbin2genind(affin.dna)
affin.pops <-  gsub( "_.*$", "", rownames(affin.gen@tab))
affin.gen@pop <- as.factor(affin.pops)
affin.scaled <- scaleGen(affin.gen,NA.method="mean",scale=F)
affin.pca <- prcomp(affin.scaled,center=F,scale=F)
affin.pc <- data.frame(affin.pca$x[,1:3])
affin.pc$sample <- rownames(affin.pc)
affin.pc$pop <- affin.pops
affin.pc$species <- rep("Eurysternus_affin",nrow(affin.pc))

# merge with sample data
affin.pc <- merge(affin.pc, localities, by.x = "sample", by.y = "ddocent_ID")
```

Plots:

![](scarab_analysis_notebook_files/figure-gfm/unnamed-chunk-21-1.png)<!-- -->

This is interesting: two extreme outliers. Let’s figure out what they
are.

``` r
# drop outliers
drop <- affin.pc[which(affin.pc$PC1>5),] 
drop <- drop$sample
drop
```

    ## [1] "1175_137" "1175_138"

Turns out these two samples were marked as potentially belonging to a
species other than *E. affin*. Seems like that’s almost certainly the
case, so let’s drop them from the dataset, and merge all our separate
data frames.

``` r
affin.gen <- affin.gen[indNames(affin.gen)!=drop[1] & indNames(affin.gen)!=drop[2]]
affin.pops <-  gsub( "_.*$", "", rownames(affin.gen@tab))
affin.gen@pop <- as.factor(affin.pops)
affin.scaled <- scaleGen(affin.gen,NA.method="mean",scale=F)
affin.pca <- prcomp(affin.scaled,center=F,scale=F)
affin.pc <- data.frame(affin.pca$x[,1:3])
affin.pc$sample <- rownames(affin.pc)
affin.pc$pop <- affin.pops
affin.pc$species <- rep("Eurysternus_affin",nrow(affin.pc))

# merge with sample data
affin.pc <- merge(affin.pc, localities, by.x = "sample", by.y = "ddocent_ID")

# merge all data frames
master.pc.df <- rbind.data.frame(satanas.pc,spec.pc,tess.pc,pod.pc,affin.pc)
```

Now let’s plot all these together with faceting.

![](scarab_analysis_notebook_files/figure-gfm/unnamed-chunk-24-1.png)<!-- -->

This still looks a lot like panmixia, but this will be useful going
forward, as we know we won’t be violating model assumptions for future
analyses (at least except for *D. speciocissimum*, which we’ll have to
deal with later).

## fineRADstructure

As a slightly more robust and sensitive approach to detecting population
structure, we’re going to estimate a covariance ancestry matrix among
individuals using
[fineRADstructure](https://github.com/millanek/fineRADstructure), which
works by estimating nearest-neighor relationships among haploytypes.
First, we’re going to drop indviduals from our .vcfs that appear to be
from cryptic species or highly divergent subspecies (as identified by
PCA above). Navigate to the `raw_data/` folder.

``` bash
vcftools --remove-indv 2150_132 --remove-indv MA2150_224 --remove-indv MA2150_226 --remove-indv MA2150_227 --remove-indv MA2150_230 --remove-indv MA2150_232 --remove-indv MA2150_233 --remove-indv YY2150_235 --vcf d_spec_filtered.FIL.recode.vcf --recode --out d_spec_filtered.strict.recode.vcf 
vcftools --remove-indv 1175_137 --remove-indv 1175_138 --vcf e_affin_filtered.FIL.recode.vcf --recode --out e_affin_filtered.strict.recode.vcf
```

Next, we need to convert our .vcfs into the haplotype format used by
fineRADstructure.

``` bash
./RADpainter hapsFromVCF ~/Dropbox/scarab_migration/raw_data/d_satanas_filtered.FIL.recode.vcf > ~/Dropbox/scarab_migration/raw_data/d_satanas_painter.txt
./RADpainter hapsFromVCF ~/Dropbox/scarab_migration/raw_data/d_spec_filtered.strict.recode.vcf > ~/Dropbox/scarab_migration/raw_data/d_spec_painter.txt
./RADpainter hapsFromVCF ~/Dropbox/scarab_migration/raw_data/d_tess_filtered.FIL.recode.vcf > ~/Dropbox/scarab_migration/raw_data/d_tess_painter.txt
./RADpainter hapsFromVCF ~/Dropbox/scarab_migration/raw_data/d_pod_filtered.FIL.recode.vcf > ~/Dropbox/scarab_migration/raw_data/d_pod_painter.txt
./RADpainter hapsFromVCF ~/Dropbox/scarab_migration/raw_data/e_affin_filtered.strict.recode.vcf > ~/Dropbox/scarab_migration/raw_data/e_affin_painter.txt
```

Next, we’re going to run the `paint` command to generate the coancestry
matrices.

``` bash
./RADpainter paint ~/Dropbox/scarab_migration/raw_data/d_satanas_painter.txt 
./RADpainter paint ~/Dropbox/scarab_migration/raw_data/d_spec_painter.txt 
./RADpainter paint ~/Dropbox/scarab_migration/raw_data/d_tess_painter.txt 
./RADpainter paint ~/Dropbox/scarab_migration/raw_data/d_pod_painter.txt 
./RADpainter paint ~/Dropbox/scarab_migration/raw_data/e_affin_painter.txt 
```

We then import these data into `R`, and manipluate the data into the
proper format for plotting with `geom_tile()`. This is a bit redundant
for each species, so I’ll put all the code here with minimal annotation.

``` r
# d. satanas

# load and format correlation matrix
satanas <- as.matrix(read.table("~/Dropbox/scarab_migration/raw_data/d_satanas_painter_chunks.out"))
colnames(satanas) <- NULL
names <- as.vector(satanas[,1])[-1]
satanas <- satanas[-1,]
satanas <- satanas[,-1]
colnames(satanas) <- names
rownames(satanas) <- names
class(satanas) <- "numeric"

# melt df
melted_satanas<- melt(satanas)
melted_satanas$pop.x <- gsub( "_.*$", "", melted_satanas$Var1) %>% as.factor()
melted_satanas$pop.y <- gsub( "_.*$", "", melted_satanas$Var2) %>% as.factor()

# d. speciocissimum

# load and format correlation matrix
spec <- as.matrix(read.table("~/Dropbox/scarab_migration/raw_data/d_spec_painter_chunks.out"))
colnames(spec) <- NULL
names <- as.vector(spec[,1])[-1]
spec <- spec[-1,]
spec <- spec[,-1]
colnames(spec) <- names
rownames(spec) <- names
class(spec) <- "numeric"

# melt df
melted_spec<- melt(spec)
melted_spec$pop.x <- gsub( "_.*$", "", melted_spec$Var1) %>% as.factor()
melted_spec$pop.y <- gsub( "_.*$", "", melted_spec$Var2) %>% as.factor()

# d tesselatum

# load and format correlation matrix
tess <- as.matrix(read.table("~/Dropbox/scarab_migration/raw_data/d_tess_painter_chunks.out"))
colnames(tess) <- NULL
names <- as.vector(tess[,1])[-1]
tess <- tess[-1,]
tess <- tess[,-1]
colnames(tess) <- names
rownames(tess) <- names
class(tess) <- "numeric"

# melt df
melted_tess<- melt(tess)
melted_tess$pop.x <- gsub( "_.*$", "", melted_tess$Var1) %>% as.factor()
melted_tess$pop.y <- gsub( "_.*$", "", melted_tess$Var2) %>% as.factor()

# d. podalirius

# load and format correlation matrix
pod<- as.matrix(read.table("~/Dropbox/scarab_migration/raw_data/d_pod_painter_chunks.out"))
colnames(pod) <- NULL
names <- as.vector(pod[,1])[-1]
pod <- pod[-1,]
pod <- pod[,-1]
colnames(pod) <- names
rownames(pod) <- names
class(pod) <- "numeric"

# melt df
melted_pod<- melt(pod)
melted_pod$pop.x <- gsub( "_.*$", "", melted_pod$Var1) %>% as.factor()
melted_pod$pop.y <- gsub( "_.*$", "", melted_pod$Var2) %>% as.factor()

# load and format correlation matrix
affin <- as.matrix(read.table("~/Dropbox/scarab_migration/raw_data/e_affin_painter_chunks.out"))
colnames(affin) <- NULL
names <- as.vector(affin[,1])[-1]
affin <- affin[-1,]
affin <- affin[,-1]
colnames(affin) <- names
rownames(affin) <- names
class(affin) <- "numeric"

# melt df
melted_affin<- melt(affin)
melted_affin$pop.x <- gsub( "_.*$", "", melted_affin$Var1) %>% as.factor()
melted_affin$pop.y <- gsub( "_.*$", "", melted_affin$Var2) %>% as.factor()
```

One issue with the data in its current form is “coancestry” takes on
different values for different species. We’ll add a statistic for
“relative coancestry,” scaled by the maximum value for each species.
We’ll also prepare the dataset for faceting.

``` r
melted_satanas$relative_coancestry <- melted_satanas$value/max(melted_satanas$value)
melted_spec$relative_coancestry <- melted_spec$value/max(melted_spec$value)
melted_tess$relative_coancestry <- melted_tess$value/max(melted_tess$value)
melted_pod$relative_coancestry <- melted_pod$value/max(melted_pod$value)
melted_affin$relative_coancestry <- melted_affin$value/max(melted_affin$value)

melted_satanas$species <- rep("dichotomius_satanas", nrow(melted_satanas))
melted_spec$species <- rep("deltochilum_speciocissimum", nrow(melted_spec))
melted_tess$species <- rep("deltochilum_tesselatum", nrow(melted_tess))
melted_pod$species <- rep("dichotomius_podalirius", nrow(melted_pod))
melted_affin$species <- rep("eurysternus_affin", nrow(melted_affin))

melted.coanc.df <- rbind.data.frame(melted_satanas,melted_spec,melted_tess,melted_pod,melted_affin)
melted.coanc.df$species <- factor(melted.coanc.df$species,levels=c("eurysternus_affin","dichotomius_podalirius",
                                                           "deltochilum_tesselatum","deltochilum_speciocissimum","dichotomius_satanas"))
```

Then, we plot our coancestry matrices:

![](scarab_analysis_notebook_files/figure-gfm/unnamed-chunk-30-1.png)<!-- -->

I left off axis labels because it made the figure too chaotic, but
essentially what we’re looking for here are contiguous chunks of similar
coancestry, indicating population genetic structure. There’s clearly
none in any of these species, providing more evidence of panmixia.

## Testing for isolation by environment

Next, we’re going to use [Bradburd et al.’s
BEDASSLE](http://www.genescape.org/bedassle.html) to estimate the
relative contribution of isolation by environment and isolation by
distance to genetic differentiaiton. For this analysis, we’ll need to
trim all our SNPs trim our SNPs for LD, which we’ll do on the raw .vcf
files using [bcftools](https://samtools.github.io/bcftools/) and
r<sup>2</sup> value of 0.1:

``` bash
bcftools +prune -l 0.1 -w 1000 d_satanas_filtered.FIL.recode.vcf -Ov -o d_satanas.LD.vcf
bcftools +prune -l 0.1 -w 1000 d_spec_filtered.strict.recode.vcf -Ov -o d_spec.LD.vcf
bcftools +prune -l 0.1 -w 1000 d_tess_filtered.FIL.recode.vcf -Ov -o d_tess.LD.vcf
bcftools +prune -l 0.1 -w 1000 d_pod_filtered.FIL.recode.vcf -Ov -o d_pod.LD.vcf
bcftools +prune -l 0.1 -w 1000 e_affin_filtered.strict.recode.vcf -Ov -o e_affin.LD.vcf
```

We need to read in locality data and use this tp calculate geographic
distance with
[fossil’s](https://cran.r-project.org/web/packages/fossil/index.html)
`earth.dist()` function. We normalize the geographic distances by
dividing by their standard deviation to help with chain mixing, saving
the stand deviation constant to later backtransform our variables:

``` r
library(adegenet)
library(BEDASSLE)
library(dplyr)
library(ggplot2)
library(broom)
library(data.table)
library(vcfR)
library(fossil)
library(viridis)
library(patchwork)
library(sp)
library(raster)
library(ecodist)
library(radiator)
library(wesanderson)

# set wd and read in locality data
setwd("/Users/ethanlinck/Dropbox/scarab_migration/")
localities <- read.csv("data/scarab_spp_master.csv")
localities$short_locality[which(localities$short_locality=="730")] <- 800
pop.loc <- cbind.data.frame(unique(localities$long),unique(localities$lat),unique(localities$short_locality))
colnames(pop.loc) <- c("long", "lat", "pop")
geodist <- earth.dist(pop.loc[c("long", "lat")], dist = FALSE)

# standardize geographic distances
geo.effect <- sd(c(geodist)) # save effect size
geodist <- geodist/sd(c(geodist))
```

We then will calculate *environmental* distances using the
[raster](https://cran.r-project.org/web/packages/raster/index.html)
package, which directly collects data from the [WorldClim climate
database](https://www.worldclim.org/). In addition to our elevational
data, we’ll use mean annual temperature and mean annual precipitation
(the bio1 and bio12 variables). Again, we’ll normalize these and save
the standard deviation constant.

``` r
# calculate environmental distance
worldclim <- getData("worldclim",var="bio",res=0.5,lon=localities$long[1],lat=localities$lat[1])
proj <- as.character(worldclim[[2]]@crs)
worldclim <- worldclim[[c(1,12)]]
names(worldclim) <- c("temp","precip")
sp1 <- SpatialPoints(localities[,c('long', 'lat')], proj4string=CRS(proj))
values <- extract(worldclim, sp1)
loc.master <- cbind.data.frame(localities, values)
loc.uniq <- cbind.data.frame(loc.master$locality, loc.master$elevation, loc.master$temp,
                             loc.master$precip) %>% distinct()
colnames(loc.uniq) <- c("population", "elevation", "temp", "precip")
rownames(loc.uniq) <- loc.uniq$population
loc.uniq <- as.data.frame(loc.uniq)
loc.uniq <- loc.uniq[,-which(names(loc.uniq) %in% c("population"))]
env.dist.all <- dist(loc.uniq, diag = TRUE, upper = TRUE)

# standardize env distances
env.effect <- sd(c(env.dist.all)) # save effect size
env.dist.all <- env.dist.all/sd(c(env.dist.all))
```

Now we have to do some serious manipulation to the format of our
genotypic data to get it into BEDASSLE. I’ll run through the whole
procedure (from data frame manipulation to the MCMC run) for *D.
satanas* in detail, and then keep it short for the remainder of the
species. (This code is also found in `scripts/bedassle.R`). First, we’ll
read our .vcf as usual, and convert it to adegenet’s genpop format,
which is pretty close to what BEDASSLE wants in that it is a summary of
allele counts across populations. We’ll set populations by grabbing the
first part of each sample ID.

``` r
# read in .vcf, turn to genpop
satanas.vcf <- read.vcfR("~/Dropbox/scarab_migration/raw_data/d_satanas.LD.vcf", convertNA=TRUE, verbose = FALSE)
satanas.gen <- vcfR2genind(satanas.vcf) 
md <- propTyped(satanas.gen, by=c("both")) # confirm we have expected amount of missing data
table(md)[1]/table(md)[2] # proportion of missing sites
```

    ##          0 
    ## 0.05375595

``` r
# table(satanas.gen@tab) # distribution of genotypes
satanas.pops <-  gsub( "_.*$", "", rownames(satanas.gen@tab)) # add pops
satanas.gen@pop <- as.factor(satanas.pops) # make factor
satanas.b <- genind2genpop(satanas.gen, satanas.pops, quiet = TRUE) # turn into gen pop
```

Next, we’ll turn this into a matrix, and delete every other column, as
the genpop format included information on both the reference and alt
alleles, while BEDASSLE only needs one.

``` r
# convert to BEDASSLE format
satanas.ac <- as.matrix(satanas.b@tab)
del <- seq(2, ncol(satanas.ac), 2) # sequence of integers to drop non-ref allele
satanas.ac <- satanas.ac[,-del] 
```

We need to create a paired matrix with sample sizes (in chromosomes,
e.g. n\*2 for our diploid beetles). We’ll set the same dimensions and
use a loop with sample size information to populate it.

``` r
# create equally sized matrix for sample sizes
satanas.n <- matrix(nrow=nrow(satanas.ac), ncol=ncol(satanas.ac))

# name our rows the same thing
rownames(satanas.n) <- rownames(satanas.ac)

# get sample size per population
sample.n <- table(satanas.gen@pop) 

# turn this into a vector
sample.sizes <- as.vector(sample.n)

# populate each row of matrix with sample sizes for pops
for(i in 1:nrow(satanas.n)){
  satanas.n[i,] <- sample.sizes[i]
}
satanas.n <- satanas.n*2 # adjust to account for loss of one allele
```

While we’re at it, we’ll calculate pairwise FST.

``` r
# calculate pairwise Fst
satanas.p.fst.all <- calculate.all.pairwise.Fst(satanas.ac, satanas.n)
satanas.p.fst.all
```

    ##             [,1]        [,2]        [,3]        [,4]        [,5]
    ## [1,]  0.00000000 -0.01389764 -0.01894285 -0.01126266 -0.02618995
    ## [2,] -0.01389764  0.00000000 -0.01970698 -0.01412796 -0.02642221
    ## [3,] -0.01894285 -0.01970698  0.00000000 -0.01690362 -0.03351763
    ## [4,] -0.01126266 -0.01412796 -0.01690362  0.00000000 -0.02334850
    ## [5,] -0.02618995 -0.02642221 -0.03351763 -0.02334850  0.00000000
    ## [6,] -0.02282667 -0.02351437 -0.02850475 -0.02160436 -0.03300329
    ## [7,] -0.04527848 -0.04887616 -0.05530750 -0.04504069 -0.06480399
    ## [8,] -0.04372206 -0.04519955 -0.04300084 -0.04005624 -0.05617999
    ##             [,6]        [,7]        [,8]
    ## [1,] -0.02282667 -0.04527848 -0.04372206
    ## [2,] -0.02351437 -0.04887616 -0.04519955
    ## [3,] -0.02850475 -0.05530750 -0.04300084
    ## [4,] -0.02160436 -0.04504069 -0.04005624
    ## [5,] -0.03300329 -0.06480399 -0.05617999
    ## [6,]  0.00000000 -0.05464144 -0.05240501
    ## [7,] -0.05464144  0.00000000 -0.08238505
    ## [8,] -0.05240501 -0.08238505  0.00000000

``` r
# look at global Fst
satanas.p.fst <- calculate.pairwise.Fst(satanas.ac, satanas.n)
satanas.p.fst
```

    ## [1] 0.01347148

Some negative values, which are an artifact of how BEDASSLE uses Weir
and Hill’s theta as our estimator but generally confirms we have no
subdivision among our populations.

Now, we need restrict our geographic distance matrix to only those
populations we’re interested in, and turn this into a data frame.

``` r
drop.pop <- c("800","925","1050","1175")

# drop levels and calc distance
pop.loc.sat <- pop.loc[!pop.loc$pop %in% drop.pop,]
droplevels(pop.loc.sat)
satanas.geo <- earth.dist(pop.loc.sat[c("long", "lat")], dist = FALSE)

# turn to vectors
sat.dist <- as.vector(satanas.geo)
sat.gen <- as.vector(satanas.p.fst.all)

# make data frame with these variables
sat.df <- cbind.data.frame(sat.dist, sat.gen)
sat.df$species <- rep("dichotomius_satanas",nrow(sat.df))
colnames(sat.df) <- c("distance", "fst", "species")
```

We’ll do the same for our environmental distance matrix, and remove
column- and rownames to prepare it for BEDASSLE.

``` r
# drop lower transect
loc.subset <- loc.uniq[-which(rownames(loc.uniq) %in% 
                                       c("CC1-800","CC2-925","CC3-1050","CC4-1175","CC1-730")),]
env.dist.upper <- dist(loc.subset, diag = TRUE, upper = TRUE)

# prep dist matrix
satanas.env <- as.matrix(env.dist.upper)
colnames(satanas.env) <- NULL
rownames(satanas.env) <- NULL
```

Finally, we run the MCMC itself. Note the various parameters we can
tweak. I’ve previously played around with short runs to tune step sizes;
because of the low genetic differentiation in the data, it’s difficult
to get the chain to mix well.

``` r
# run MCMC for 10K gens
MCMC(   
  counts = satanas.ac,
  sample_sizes = satanas.n,
  D = geodist,  # geographic distances
  E = env.dist.all,  # environmental distances
  k = nrow(satanas.ac), loci = ncol(satanas.ac),  # dimensions of the data
  delta = 0.0001,  # a small, positive, number
  aD_stp = 0.075,   # step sizes for the MCMC
  aE_stp = 0.05,
  a2_stp = 0.025,
  thetas_stp = 0.2,
  mu_stp = 0.35,
  ngen = 2000000,        # number of steps (2e6)
  printfreq = 10000,  # print progress (10000)
  savefreq = 1e5,     # save out current state
  samplefreq = 250,     # record current state for posterior (2000)
  prefix = "/Users/ethanlinck/Dropbox/scarab_migration/bedassle/satanas_",   # filename prefix
  continue=FALSE,
  continuing.params=FALSE)
```

Once this has run, we’re going to perform some basic examination of how
well the chain performed, using code and the procedure from [Peter
Ralph’s BEDASSLE
tutorial](http://petrelharp.github.io/popgen-visualization-course/).
I’ll show the output (after dropping the first 25% of the chain as
burnin) so you can see the different objects we have to play with.

    ##  [1] "last.params"   "LnL_thetas"    "LnL_counts"    "LnL"          
    ##  [5] "Prob"          "a0"            "aD"            "aE"           
    ##  [9] "a2"            "beta"          "samplefreq"    "ngen"         
    ## [13] "a0_moves"      "aD_moves"      "aE_moves"      "a2_moves"     
    ## [17] "thetas_moves"  "mu_moves"      "beta_moves"    "aD_accept"    
    ## [21] "aE_accept"     "a2_accept"     "thetas_accept" "mu_accept"    
    ## [25] "aD_stp"        "aE_stp"        "a2_stp"        "thetas_stp"   
    ## [29] "mu_stp"

![](scarab_analysis_notebook_files/figure-gfm/unnamed-chunk-41-1.png)<!-- -->![](scarab_analysis_notebook_files/figure-gfm/unnamed-chunk-41-2.png)<!-- -->![](scarab_analysis_notebook_files/figure-gfm/unnamed-chunk-41-3.png)<!-- -->![](scarab_analysis_notebook_files/figure-gfm/unnamed-chunk-41-4.png)<!-- -->![](scarab_analysis_notebook_files/figure-gfm/unnamed-chunk-41-5.png)<!-- -->

While it didn’t quite converge, it’s probably as close as it’s going to
get, and we can see the posterior is heavily weighted towards the aE/aD
ratio we expect, e.g. very little to very little (because there is very
little differentiation). We’ll save this in a data frame we’ll later use
to make a nicer looking figure, and get the back-transformed values from
our saved standard deviation constants.

``` r
# convert for ggplotting
sat.ratio <- aE/aD %>% as.data.frame()
sat.bed.df <- cbind.data.frame(sat.ratio, rep("dichotomius_satanas", nrow(sat.ratio)))
colnames(sat.bed.df) <- c("ratio","species")

aD <- aD/geo.effect
aE <- aE/env.effect
mean(aE[-c(1:2000)]/aD[-c(1:2000)]) #0.0001822692
```

    ## [1] 0.002477588

``` r
sd(aE[-c(1:2000)]/aD[-c(1:2000)]) #6.475261e-05
```

    ## [1] 0.0008351278

``` r
aD <- aD[-c(1:2000)] # drop burnin again
aE <- aE[-c(1:2000)] 
```

Phew\! Only four more species to go. As I mentioned above, I’m going to
skip the details of these runs and get to what we’re interested in,
which is the posterior probabilities of the aE/aD ratio for all of them.
To do this, I’m going to load the completed MCMC outputs from all
species, and merge the ratio into a data frame.

Now, let’s check it out.

![](scarab_analysis_notebook_files/figure-gfm/unnamed-chunk-44-1.png)<!-- -->

Again, we’re having some issues with convergence, but the weight of the
distributions is very low. Mountain passes *aren’t* higher for these
beetles, at least within their elevation range\!

# Estimating Wright’s neighborhood size and theta

Next, we’re going to use Rousset’s (1997) method for estimating Wright’s
neighborhood size as the slope of a linear regression of fst/(1-fst) by
the natural log of geographic disance. We already calculated pairwise
fst values and distances with for all species in our script
`bedassle.R`, and now we’ll load that, drop negative distances, and
calculate the natural log of geographic distance:

``` r
fst.df <- read.csv(file="~/Dropbox/scarab_migration/data/fst_dist_species.csv")[-1]
fst.df <- fst.df[fst.df$distance>0,]
fst.df$fst[fst.df$fst<0] <- 0
fst.df$log_distance <- log2(fst.df$distance)
fst.df$fst.adj <- fst.df$fst/(1-fst.df$fst)
head(fst.df)
```

    ##    distance fst             species log_distance fst.adj
    ## 2 0.1157304   0 dichotomius_satanas   -3.1111603       0
    ## 3 0.2199334   0 dichotomius_satanas   -2.1848617       0
    ## 4 0.5060120   0 dichotomius_satanas   -0.9827566       0
    ## 5 0.6362566   0 dichotomius_satanas   -0.6523193       0
    ## 6 0.4941098   0 dichotomius_satanas   -1.0170966       0
    ## 7 2.4938052   0 dichotomius_satanas    1.3183488       0

Now, we can subset each species out of this dataframe

``` r
# d. satanas 
sat.fst <- fst.df[fst.df$species=="dichotomius_satanas",]
head(sat.fst)
```

    ##    distance fst             species log_distance fst.adj
    ## 2 0.1157304   0 dichotomius_satanas   -3.1111603       0
    ## 3 0.2199334   0 dichotomius_satanas   -2.1848617       0
    ## 4 0.5060120   0 dichotomius_satanas   -0.9827566       0
    ## 5 0.6362566   0 dichotomius_satanas   -0.6523193       0
    ## 6 0.4941098   0 dichotomius_satanas   -1.0170966       0
    ## 7 2.4938052   0 dichotomius_satanas    1.3183488       0

``` r
sat.mod <- lm(fst.adj ~ log_distance, sat.fst)
1/as.vector(sat.mod$coefficients[2]) #[1] Inf
```

    ## [1] Inf

``` r
# d. spec
spec.fst <- fst.df[fst.df$species=="deltochilum_speciocissimum",]
spec.mod <- lm(fst.adj ~ log_distance, spec.fst)
1/as.vector(spec.mod$coefficients[2]) 
```

    ## [1] 13542.25

``` r
# d. tess
tess.fst <- fst.df[fst.df$species=="deltochilum_tesselatum",]
tess.mod <- lm(fst.adj ~ log_distance, tess.fst)
1/as.vector(tess.mod$coefficients[2]) 
```

    ## [1] -360.0897

``` r
# d. pod
pod.fst <- fst.df[fst.df$species=="dichotomius_podalirius",]
pod.mod <- lm(fst.adj ~ log_distance, pod.fst)
1/as.vector(pod.mod$coefficients[2]) 
```

    ## [1] -Inf

``` r
# e. affin
affin.fst <- fst.df[fst.df$species=="eurysternus_affin",]
affin.mod <- lm(fst.adj ~ log_distance, affin.fst)
1/as.vector(affin.mod$coefficients[2])
```

    ## [1] -68.55046

In all cases, the slope of our regression is negative, which results in
an estimate of Wright’s NS which is more or less infinite, and
consistent with panmixia (at least as inferred from our restricted
sampling.) Not terrible surprising, but good to know. Let’s plot these
as well:

A natural follow up question is trying to understand how genetic
diversity changes across elevation in our species, as this can tell us
something about the nature of their elevational range limits, e.g. that
they either are or are not affected by declining habitat quality or edge
effects. We’ll use the
[pegas](https://cran.r-project.org/web/packages/pegas/index.html)
package to estimate theta (the product of the effective population size
and the neutral mutation rate) using mean homozygosity from gene
frequencies.

We’ll have to read in our .vcf files again, manipulate them into the
right format, and run a loop over each RAD locus to generate more or
less independent estimates from across the genome. We’re going to use
our non-LD trimmed dataset, as having more SNPs per locus will improve
accuracy for each point. Here’s an example from *D. satanas*; I’ll leave
the remainder for inspection in the `stats.R` file, and load the
processed data matrix to save computational time.

``` r
# d. satanas
satanas.vcf <- read.vcfR("raw_data/d_satanas_filtered.FIL.recode.vcf", convertNA=TRUE)
satanas.gen <- vcfR2genind(satanas.vcf) 
satanas.pops <-  gsub( "_.*$", "", rownames(satanas.gen@tab)) # add pops
satanas.gen@pop <- as.factor(satanas.pops) 
sat.S <- as.loci(satanas.gen) # turn into locus object for pegas 

satlist <- list() # empty list to fill with results
pops <- levels(sat.S$population)
for(i in pops){ # loop over populations and estimate theta for each allele 
  tmp <- sat.S[which(sat.S$population==i),]
  sum.tmp <- summary(tmp)
  tmp.theta <- as.vector(sapply(sum.tmp, function(x) theta.h(x$allele)))
  tmp.df <- cbind.data.frame(tmp.theta, rep(i, length(tmp.theta)))
  colnames(tmp.df) <- c("theta", "population")
  satlist[[i]] <- tmp.df
}

sat.df <- do.call(rbind, satlist) # create dataframe
sat.df <- sat.df[sat.df$theta>0,]
sat.df$species <- rep("dichotomius_satanas",nrow(sat.df)) # prepare to merge with other results eventually
colnames(sat.df) <- c("theta", "population","species")
```

There are a few tweaks for the idiosyncrasies of other species, but
that’s basically it. Let’s load the final dataset from file:

``` r
master.df <- read.csv("~/Dropbox/scarab_migration/data/master.theta.df.csv")[-1]
head(master.df)
```

    ##       theta population             species elevation
    ## 1 0.3818161       1575 dichotomius_satanas      1575
    ## 2 0.3780009       1575 dichotomius_satanas      1575
    ## 3 0.3818161       1575 dichotomius_satanas      1575
    ## 4 0.3818161       1575 dichotomius_satanas      1575
    ## 5 0.3818161       1575 dichotomius_satanas      1575
    ## 6 0.3818161       1575 dichotomius_satanas      1575

We can see columns for the theta value of each species, the name of the
population, and an elevation. Each row is an estimate from a different
RAD locus. Let’s plot these, after doing some manipulations to clean up
population name discrepancies:

![](scarab_analysis_notebook_files/figure-gfm/unnamed-chunk-50-1.png)<!-- -->

We can see that theta doesn’t really change across elevation. We’ll back
this up with some statistics, using mixed models to ask whether distance
from range center affects genetic diversity, with population as a random
effect:

``` r
library(lme4)

# run from file
master.df <- read.csv("~/Dropbox/scarab_migration/data/master.theta.df.csv")
master.df$species <- as.factor(master.df$species)
master.df$species <- factor(master.df$species,levels=c("eurysternus_affin","dichotomius_podalirius",
                                                       "deltochilum_tesselatum","deltochilum_speciocissimum","dichotomius_satanas"))

# subset theta df 
affin.mast <- master.df[master.df$species=="eurysternus_affin",]
pod.mast <- master.df[master.df$species=="dichotomius_podalirius",]
tess.mast <- master.df[master.df$species=="deltochilum_tesselatum",]
spec.mast <- master.df[master.df$species=="deltochilum_speciocissimum",]
sat.mast <- master.df[master.df$species=="dichotomius_satanas",]

# calculate absolute value of distance from mean sampling elevation
affin.mid <- mean(affin.mast$elevation)
affin.mast$limit_score <- abs(affin.mast$elevation-affin.mid)
pod.mid <- mean(pod.mast$elevation)
pod.mast$limit_score <- abs(pod.mast$elevation-pod.mid)
tess.mid <- mean(tess.mast$elevation)
tess.mast$limit_score <- abs(tess.mast$elevation-tess.mid)
spec.mid <- mean(spec.mast$elevation)
spec.mast$limit_score <- abs(spec.mast$elevation-spec.mid)
sat.mid <- mean(sat.mast$elevation)
sat.mast$limit_score <- abs(sat.mast$elevation-sat.mid)

# mixed models to see whether distance from range limit affects values of theta
affin.theta.mod <- lmer(theta ~ limit_score + (1 | population), affin.mast)
null.affin.mod <-  lmer(theta ~ (1 | population), affin.mast)
anova(affin.theta.mod,null.affin.mod)  #X2=2.309(1), p=0.1286
```

    ## Data: affin.mast
    ## Models:
    ## null.affin.mod: theta ~ (1 | population)
    ## affin.theta.mod: theta ~ limit_score + (1 | population)
    ##                 Df     AIC     BIC logLik deviance Chisq Chi Df Pr(>Chisq)
    ## null.affin.mod   3 -1574.8 -1559.9 790.41  -1580.8                        
    ## affin.theta.mod  4 -1575.1 -1555.2 791.56  -1583.1 2.309      1     0.1286

``` r
pod.theta.mod <- lmer(theta ~ limit_score + (1 | population), pod.mast)
null.pod.mod <-  lmer(theta ~ (1 | population), pod.mast)
anova(pod.theta.mod,null.pod.mod)  #X2=2.276(1), p=0.1314
```

    ## Data: pod.mast
    ## Models:
    ## null.pod.mod: theta ~ (1 | population)
    ## pod.theta.mod: theta ~ limit_score + (1 | population)
    ##               Df     AIC     BIC logLik deviance Chisq Chi Df Pr(>Chisq)
    ## null.pod.mod   3 -5578.1 -5559.4 2792.1  -5584.1                        
    ## pod.theta.mod  4 -5578.4 -5553.4 2793.2  -5586.4 2.276      1     0.1314

``` r
tess.theta.mod <- lmer(theta ~ limit_score + (1 | population), tess.mast)
null.tess.mod <-  lmer(theta ~ (1 | population), tess.mast)
anova(tess.theta.mod,null.tess.mod)  #X2=0.9443(1), p=0.3312
```

    ## Data: tess.mast
    ## Models:
    ## null.tess.mod: theta ~ (1 | population)
    ## tess.theta.mod: theta ~ limit_score + (1 | population)
    ##                Df    AIC    BIC logLik deviance  Chisq Chi Df Pr(>Chisq)
    ## null.tess.mod   3 -23466 -23443  11736   -23472                         
    ## tess.theta.mod  4 -23465 -23434  11736   -23473 0.9443      1     0.3312

``` r
spec.theta.mod <- lmer(theta ~ limit_score + (1 | population), spec.mast)
null.spec.mod <-  lmer(theta ~ (1 | population), spec.mast)
anova(spec.theta.mod,null.spec.mod)  #X2=2.045(1), p=0.1527
```

    ## Data: spec.mast
    ## Models:
    ## null.spec.mod: theta ~ (1 | population)
    ## spec.theta.mod: theta ~ limit_score + (1 | population)
    ##                Df    AIC    BIC logLik deviance Chisq Chi Df Pr(>Chisq)
    ## null.spec.mod   3 -98616 -98589  49311   -98622                        
    ## spec.theta.mod  4 -98616 -98580  49312   -98624 2.045      1     0.1527

``` r
sat.theta.mod <- lmer(theta ~ limit_score + (1 | population), sat.mast)
null.sat.mod <-  lmer(theta ~ (1 | population), sat.mast)
anova(sat.theta.mod,null.sat.mod)  #X2=0.9191(1), p=0.3377
```

    ## Data: sat.mast
    ## Models:
    ## null.sat.mod: theta ~ (1 | population)
    ## sat.theta.mod: theta ~ limit_score + (1 | population)
    ##               Df    AIC    BIC logLik deviance  Chisq Chi Df Pr(>Chisq)
    ## null.sat.mod   3 -18616 -18594 9311.0   -18622                         
    ## sat.theta.mod  4 -18615 -18586 9311.5   -18623 0.9191      1     0.3377

In all cases, there’s no significant change in diversity.

Taking all these results together, the last outstanding question is
whether elevational ranges are at equilibrium, something this results
certainly suggest and something that would tell us about the limits of
Janzen’s predictions in this dataset. We’ll test this with demographic
modeling using approximate Bayesian computation (ABC).

First, we need to createa a .vcf file without distortions to the SFS.
We’ll return to the raw .vcf for each species, and again apply a
series of filtering steps, but avoid anything that affects the SFS,
e..g. minor allele frequency filters.

``` bash
vcftools --vcf d_satanas.raw.vcf --max-missing 0.75 --minQ 30 --minDP 3 --min-meanDP 5 --remove lowDP.indv --recode --recode-INFO-all --out d.satanas.abc 
bash dDocent_filters d.satanas.abc.recode.vcf d.satanas.abc

vcftools --vcf d_spec.raw.vcf --max-missing 0.75 --minQ 30 --minDP 3 --min-meanDP 5 --remove lowDP.indv --recode --recode-INFO-all --out d.spec.abc 
bash dDocent_filters d.spec.abc.recode.vcf d.spec.abc

vcftools --vcf d_tess.raw.vcf --max-missing 0.75 --minQ 30 --minDP 3 --min-meanDP 5 --remove lowDP.indv --recode --recode-INFO-all --out d.tess.abc 
bash dDocent_filters d.tess.abc.recode.vcf d.tess.abc

vcftools --vcf d_pod.raw.vcf --max-missing 0.75 --minQ 30 --minDP 3 --min-meanDP 5 --remove lowDP.indv --recode --recode-INFO-all --out d.pod.abc 
bash dDocent_filters d.pod.abc.recode.vcf d.pod.abc

vcftools --vcf e_affin.raw.vcf --max-missing 0.75 --minQ 30 --minDP 3 --min-meanDP 5 --remove lowDP.indv --recode --recode-INFO-all --out e.affin.abc 
bash dDocent_filters e.affin.abc.recode.vcf e.affin.abc

mkdir abc
cp d_satanas/filtering/*.abc.FIL.recode.vcf abc/
cp d_speciocissimum/filtering/*.abc.FIL.recode.vcf abc/
cp d_tesselatum/filtering/*.abc.FIL.recode.vcf abc/
cp d_podalirius/filtering/*.abc.FIL.recode.vcf abc/
cp e_affin/filtering/*.abc.FIL.recode.vcf abc/
```

Now that we have appropriately treated .vcf files, we’re going to test
whether our five species have undergone recent range expansion
(e.g. upslope or downslope) using an [approximate Bayesian computation
(ABC) approach](https://www.genetics.org/content/162/4/2025). We’ll use
the [coala](https://github.com/statgenlmu/coala) (coalescent simulator
wrapper) and
[abc](https://cran.r-project.org/web/packages/abc/index.html) R packages
to do this, and manually calculate the site frequency spectrum for each
species to use as a summary statistic. As with BEDASSLE, I’m only going
to show the code for the first species as the rest will be repetitive,
but it’s all available in `scripts/abc.R`.

``` r
library(coala)
library(abc)

# calculate SFS from data
satanas.vcf <- read.vcfR("~/Dropbox/scarab_migration/raw_data/d.satanas.abc.FIL.recode.vcf", convertNA=TRUE)
satanas.gen <- vcfR2genlight(satanas.vcf) 
sat.sfs <- table(glSum(satanas.gen))
sat.sfs <- as.vector(sat.sfs[1:98])

# define two models
sat.null.model <- coal_model(99, 50, 3) +
  feat_mutation(par_prior("theta", runif(1, 1, 6))) +
  sumstat_sfs()

sat.growth.model <- coal_model(99, 50, 3) +
  feat_mutation(par_prior("theta", runif(1, 1, 6))) +
  feat_growth(par_prior("r", runif(1, 0, 1))) +
  sumstat_sfs()

# simulate data
sat.null.sim <- simulate(sat.null.model, nsim = 100000, seed = 69)
sat.growth.sim <- simulate(sat.growth.model, nsim = 100000, seed = 32)

# create params and sumstts for abc
sat.null.param <- create_abc_param(sat.null.sim, sat.null.model)
sat.null.sumstat <- create_abc_sumstat(sat.null.sim, sat.null.model)
sat.growth.param <- create_abc_param(sat.growth.sim, sat.growth.model)
sat.growth.sumstat <- create_abc_sumstat(sat.growth.sim, sat.growth.model)

# look at posterior distributions of params
sat.null.post <- abc(sat.sfs, sat.null.param, sat.null.sumstat, 0.05, method = "rejection")
sat.growth.post <- abc(sat.sfs, sat.growth.param, sat.growth.sumstat, 0.05, method = "rejection")

# prep data for model 
sat.sumstat.merge <- rbind(sat.null.sumstat, sat.growth.sumstat)
sat.index <- c(rep("null",100000),rep("growth",100000))

# check ability to distinguish models
sat.cv.modsel <- cv4postpr(sat.index, sat.sumstat.merge, nval=5, tols=c(.05,.1), method="rejection")
summary(sat.cv.modsel)

# model test
sat.model <- postpr(target=sat.sfs, sumstat=sat.sumstat.merge, index=as.vector(sat.index), tol=.05, method="rejection")
summary(sat.model)

# prep dataframe
sat.growth.plot <- as.data.frame(sat.growth.post$unadj.values)
sat.growth.theta <- cbind.data.frame(sat.growth.plot$theta, rep("theta", nrow(sat.growth.plot)))
colnames(sat.growth.theta) <- c("value","param")
sat.growth.r <- cbind.data.frame(sat.growth.plot$r, rep("r", nrow(sat.growth.plot)))
colnames(sat.growth.r) <- c("value","param")
sat.growth.df <- rbind.data.frame(sat.growth.theta,sat.growth.r)
sat.growth.df$species <- rep("dichotomius_satanas", nrow(sat.growth.df))
```

We’ve calculated Bayes Factors for the support of different model
comparisons (e.g. null model:exponential growth; there are four in
total), and the posterior probabilities of parameter values for both.

Let’s read the results for all species, and plot them all together.
We’ll use our usual species order. First, the dataframe manipulation.

Now, the plotting:

![](scarab_analysis_notebook_files/figure-gfm/unnamed-chunk-55-1.png)<!-- -->![](scarab_analysis_notebook_files/figure-gfm/unnamed-chunk-55-2.png)<!-- -->

In all but one compairson (for *E. affin*; you read these matrices as
the Bayes Factor value for model comparisons from the x axis to the y
axis), the null model is much more strongly supported. And when we force
a model of exponential growth, the growth parameter’s (r) value is very
low. In other words, it seems like elevational ranges are at
equilibrium.
