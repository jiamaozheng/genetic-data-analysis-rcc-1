# Analysis of Genetic Data 1:<br>Exploring structure of genetic data using PCA

This episode is divided into two parts. In the **first part**, we will
observe how genetic polymorphism data---specifically genotype samples
from human subjects---are commonly stored in computer files. We will
also see how these data can be represented as a **matrix**.

In the **second part**, we will explore **principal component
analysis** (PCA), a widely used statistical technique for exposing
hidden patterns, or "structure", in genetic data. In these exercises,
we will discover that much of the structure uncovered by PCA can be
connected to human history and geography.

For these explorations, we will use genotype data from [Phase 3 of the
1000 Genomes Project](http://dx.doi.org/10.1038/nature15393), an
international scientific effort to comprehensively study global human
genetic variation and create a commmunity resource for genetics
research. These data have been made available to the public
[here](http://www.1000genomes.org/data). To reproduce the steps taken
to retrieve and prepare the 1000 Genomes data for this workshop, see
[here](../extras/1kg.md).

### A. Storing genetic data in computer files

Let's start by inspecting the file `1kg_test.ped` which you copied to
the data folder. This file contains the genotype data for 29 samples at
156,923 SNPs.

```bash
cd ~/git/gda1/data
less -S 1kg_test.ped
```

The program `less` with the `-S` flag is well suited for viewing very
large text files because the file can be viewed without having to read
the entire file. You can scroll up-down and left-right using the
arrow keys on your keyboard.

A line of this file stores the genotype data for a single sample; each
a pair of letters after the first 6 columns specifies the genotype at
a given SNP (a nucleotide on each chromosome copy). Some genotype
entries are `0 0`, meaning that the genotype is unavailable.

:ledger: This file is over 18 MB in size just to store the genotypes
for 29 samples. Why is this text format an inefficient way to store
genotype data? How could you store the data more efficiently?

### B. Representing genetic data as a matrix

In order to apply numerical analyses to the genotype data, we need to
be able to represent these data in a numerical form. Here we will use
PLINK to convert the genotype data for test samples into a matrix.

First we need to convert the genotype data (stored in a PLINK file) to
a p x n matrix, where p is the number of genetic markers (SNPs) and n
is the number of samples. 

```bash
module load plink/1.90
cd ~/git/gda1/data
plink --file 1kg_test --recode A-transpose spacex --out 1kg_test
less -S 1kg_test.traw
```

Each line of the `1kg_test.traw` text file, after the header line,
stores a row of the p x n matrix, and each (space-delimited) column of
the text file, after the first 6 columns, stores a column of the p x n
matrix. Observe that all the matrix entries are either 0, 1 or 2 (or
'NA' for "not available", or "missing"). Therefore, we now have a
fully numeric representation of the genotype data.

:ledger: Why is a single number (0, 1 or 2) sufficient to represent
the genotype? How much more efficient is this representation compared
to `1kg_test.ped`?

### C. Computing PCs from genetic data using MATLAB

PCA is an important numerical technique for data analysis in many
scientific fields, and it would be inappropriate to attempt to explain
PCA within this workshop, let alone motivate its importance for
genetics research. One simple way to view PCA is as a dimensionality
reduction technique for embedding multi-dimensional data in a small
number of dimensions (e.g., for plotting samples in 2
dimensions). Here we will gain some non-mathematical intuition for PCA
by illustrating its use in genetic data.

Several software tools have been developed for efficiently computing
principal components (PCs) from large genetic data sets, such as
[EIGENSOFT](https://www.hsph.harvard.edu/alkes-price/software). Instead
of using a specialized program, here we use MATLAB to demonstrate that
PCA is easily computed using well established numerical techniques; in
particular, PCA is studied as the singular value decomposition (SVD)
of a matrix, so we will use a
[fast MATLAB implementation of the Lanczos algorithm developed by Jie Chen at IBM](https://jie-chen-ibm.appspot.com/software.html)
called [svdk](../code/svdk.m) to compute the the first *k* singular
values of the genotype matrix and their associated singular vectors.

*Note:* To proceed from this point you may need more memory than was
initially suggested in [Setup](01-setup.md). It seems that requesting 8
GB of memory is sufficient for these computations.

Before computing the SVD in MATLAB, we first generate
`1kg_train.traw`, then convert it to a binary format that is
convenient for loading into MATLAB.

```bash
module load matlab/2016a
cd ~/git/gda1/data
plink --bfile 1kg_train --recode A-transpose spacex --out 1kg_train
cd ../code
matlab -nosplash -nodesktop
```

In MATLAB, run the data conversion script by entering:

```MATLAB
traw2mat
```

This MATLAB script creates a new file `1kg_train.mat` in the data
folder. Now that we have the genotype data in a convenient format for
MATLAB, we can perform the computation. This is accomplished using the
MATLAB [geno_pca.m](../code/geno_pca.m) script. Again in MATLAB, simply
enter:

```MATLAB
geno_pca
```

The key step in this script is the line that uses the
[svdk algorithm](../code/svdk.m) to compute the largest *k* singular
values and the associated singular vectors:

```MATLAB
[U S R] = svdk(X,k);
```

The console output from running this script should look something like
this:

<pre>(1) Loading genotype matrix from .mat file.
Loaded a 2289 x 156923 matrix.
(2) Centering columns of genotype matrix.
(3) Calculating first 10 PCs.
Computation took 1.41 minutes.
(4) Saving rotation matrix to text file.
(5) Projecting samples onto the principal components.
(6) Saving PCs to text file.
(7) Saving mean genotypes to text file.
</pre>

Note that this algorithm is *extremely fast*; it only takes a few
minutes to compute the SVD for the 2289 x 156923 genotype data matrix.

This script creates two files in the results folder:

1. `1kg_train_pcs.txt`, a text file storing the mapping of the samples
onto the first k PCs. 

2. `1kg_train_rot.txt`, the p x k "rotation" matrix which we will use
to project new genotype samples onto the PCA embedding in the next
episode.

:white_check_mark: You have generated all the PCA result files that we
will use for the rest of this episode and the next. Take a moment to
inspect these files using `less -S`.

### D. Visualizing PCs in R

After having done all the work to represent the genotype data as a
matrix, and compute a low-dimensional representation of this matrix
using PCA (equivalently, SVD), in this section we will generate
PC-based visualizations of the data.

First, start up the R environment:

```bash
cd ~/git/gda1
R --no-save
```

In this next code block, we load the PCA results into R.

```R
# Load the "plotpca" function and other useful functions.
library(ggplot2)
source("code/read.data.R")
source("code/plotpca.R")

# Load the expert-provided population labels as a data frame.
panel <- read.1kg.labels("data/omni_samples.20141118.panel")
print(head(panel))

# Load the genotype samples projected onto the PCs. This is a data
# frame in which the first column gives the sample ids, and the
# remaining columns give the projection of the samples onto the
# computed PCs.
pc <- read.pcs("results/1kg_train_pcs.txt")
print(head(pc),digits = 4)

# Merge the two tables; we now have an additional column giving the
# population label.
pc <- merge(panel,pc)
pc <- transform(pc,id = as.character(id))
print(head(pc),digits = 4)
```

After completing these steps, in our R environment we now have a data
frame `pc` containing the PCA results and the population labels for
all the genotype samples in the 1000 Genomes data set. Entering
`print(summary(pc$pop))` summarizes the population label counts.  For
the meaning of these population labels, refer to
[these guidelines](http://catalog.coriell.org/1/NHGRI/About/Guidelines-for-Referring-to-Populations).

We have designed an R function `plotpca` (see
[plotpca.R](code/plotpca.R) for the source code) that generates a 2-d
plot from the PCA results. For example, the following lines will plot
the samples according to their projection onto PCs 1 and 2.

```R
# Optionally, create a new graphics device. Note this doesn't work
# in RStudio.
dev.new(height = 6,width = 8)

# Plot the samples according to their projection onto PCs 1 and 2; the
# colour and shape of the samples are chosen according to the
# expert-provided population labels provided in the "pop" column.
print(plotpca(pc,1,2))
```

:blue_book: Compare your plot against Supplementary Figure 4 of the
[1000 Genomes paper from 2012](http://dx.doi.org/10.1038/nature11632). How
well do these plots agree?

:ledger: How does this visualization inform us about global genetic
diversity (or relative lack of genetic diversity) and its relationship
to geography and/or demographic trends? How would you explain in a
concise, non-technical way the main demographic trends, or *population
structure*, captured by PCs 1 and 2?

:orange_book: Which results match your expectations, and which do you
find more surprising?

:pushpin: To zoom in on certain portions of the plot, use the `zoom`
argument to `plotpca` (see the comments at the top of
[plotpca.R](code/plotpca.R) for details); for example,
`print(plotpca(pc,1,2,zoom = c(-40,10,10,70)))`.

:pencil2: Investigate these same questions for PCs 3 and 4.

:white_check_mark: In [the next episode](03-pca-project.md), we will
demonstrate the power of these same numerical techniques for making
predictions from genetic data.
