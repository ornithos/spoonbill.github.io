---
layout: post
title: nectr: Exploratory Clustering
---
A number of years ago I spent some time researching clustering algorithms due to a growing dissatisfaction with k-means - the favourite clustering algorithm of hackers. During this time I was performing a lot of exploratory analysis, and I was hoping to understand the regions of high density within my dataset for a customer segmentation. Of course k-means cannot be trusted with such a task; while it may capture some elements of the underlying density, it is just as happy creating a relatively arbitrary partition. Further, if the true density is not given to spherical (or more accurately voronoi cell) clusters, then k-means will also fail as an accurate description of the data.

While non-parametric alternatives exist such as DBSCAN or mean-shift, these are highly sensitive to parameter specification, and higher dimensions suffer from quadratic time. Spectral clustering takes a slightly different approach by acting on the graph Laplacian, but unless approximated is at least quadratic, and one must specify the number of clusters in advance. The algorithm I chose is one called TURN-RES from a [paper](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.7.1966&rep=rep1&type=pdf) by Foss \& Za¨ıane in 2002. The main advantages are that it is a non parametric approach that is (sort-of) parameterless and runs in `O(nlogn)` time. It approximates density estimation by quantizing the space into discrete grid, assigning neighbours to each datapoint in all axis-oriented direction and calculating the number of neighbours within a given distance. For a few unclear details I have taken my best guess, but otherwise the implementation of TURN-RES follows the Foss paper.

## TURN-RES
Two complementary algorithms are proposed in the Foss paper, TurnCut and TURN-RES. Is TURN an acronym? Who knows. Your guess is as good as mine. Anyway, TURN-RES is the workhorse clustering algorithm which contains a resolution parameter. TurnCut is a line-search like algorithm which seeks to find the 'elbow' or 'knee' of various metrics associated with a clustering result. While the results for the datasets used were apparently strong, I have not found this to be so useful in certain real world datasets. This is in part due to the subjectivity of the task and often a continuum of plausible results over increasing resolution. But perhaps more importantly, one often finds that different clusters have different natural resolutions and a global resolution parameter can never capture all of these. An implementation of TurnCut is not available in the package, instead I have chosen to make the cluster discovery an exploratory process. The clustering objective is not (ultimately) well-posed and here it is the analyst's job to iteratively discover the best cluster assignment.

1) 2D data
2) Quantize
3) Order horizontally and glow neighbours
4)   "   vertically " " "
5) Show connections between datapoints

## nectr: Non-Parametric Exploratory Clustering using TURN-RES
![dead](https://github.com/spoonbill/spoonbill.github.io/blob/master/images/clsTurnRes.gif "steps of TURN-RES")
The R-package is called `nectr` and can be found [here](https://github.com/spoonbill/nectr). The following is a simple demonstration of the algorithm on some dummy data. First, we create a 5 cluster fake dataset over 3 dimensions according to the following R script:

```R
library(MASS)
n <- c(175, 60, 50, 135, 105) * 1000
noise.n <- 75000

mu <- list()
mu[[1]] <- c(4,6,3)
mu[[2]] <- c(1,0,-1)
mu[[3]] <- c(-4,-1,-2)
mu[[4]] <- c(2,-5,-4)
mu[[5]] <- c(-3,3,5)

sigma <- list()
sigma[[1]] <- matrix(c(2,0.2,0.8,0.2,1,0.1,0.8,0.1,1),3,3)
sigma[[2]] <- matrix(c(1,0,-1,0,2,0.4,-1,0.4,3),3,3)
sigma[[3]] <- matrix(c(3,0,0,0,0.3,0,0,0,2),3,3)
sigma[[4]] <- matrix(c(2,1.5,1,1.5,2,1,1,1,2),3,3)
sigma[[5]] <- diag(3)*1.8

fakeData <- as.data.frame(rbind(mvrnorm(n[1], mu[[1]], sigma[[1]]), mvrnorm(n[2], mu[[2]], sigma[[2]]),
                            mvrnorm(n[3], mu[[3]], sigma[[3]]), mvrnorm(n[4], mu[[4]], sigma[[4]]), 
                            mvrnorm(n[5],mu[[5]], sigma[[5]]), matrix(runif(noise.n*3,-10,10), noise.n , 3)))
names(fakeData) <- c("X", "Y", "Z")
```

We have also added some uniformly distributed noise to make the problem slightly more challenging. The dataset is shown below - we have n = 600,000 datapoints (not all shown!), and d = 3 dimensions.

<img src="https://github.com/spoonbill/spoonbill.github.io/blob/master/images/clusters2.png" width="150") <img src="https://github.com/spoonbill/spoonbill.github.io/blob/master/images/clusters1.png" width="150")
The core algorithm TURN-RES requires one parameter - the quantization resolution. The lower the resolution, the more neighbours a datapoint has, which increases the cluster sizes. Since the algorithm is fairly fast, we can scan over multiple resolutions fairly quickly to build a multiresolution cluster object.

```R
library(nectr)
multires <- clsMRes(fakeData, keep = TRUE)
```
![alt text](https://github.com/spoonbill/spoonbill.github.io/blob/master/images/Rmultires.PNG "R output")
The clsMRes function iteratively reduces the resolution until no clusters are visible, and then increases until most datapoints are put in the same cluster. The `keep = TRUE` argument tells the function to keep all of the cluster assignments generated by the various iterations of TURN-RES. We may then use these to build a hierachical picture of how the clusters agglomerate. This took about 30 seconds on a 2.7GHz i5 (single) processor.

We can inspect the agglomeration schedule, that is, how the clusters agglomerate as the scale is increased using the command
```R
plot(multires)
```

![alt text](https://github.com/spoonbill/spoonbill.github.io/blob/master/images/agglom.png "plot clsMR object")


```R
plot(multires, c(8,12,13))
```
![alt text](https://github.com/spoonbill/spoonbill.github.io/blob/master/images/pcp8_12_13.png "Princpal Component Plot")


```R
plot(multires,6:10)
```

![alt text](https://github.com/spoonbill/spoonbill.github.io/blob/master/images/pcp_6_10.png "Parallel Coordinate Plot")

```R
gmmSpec <- clsSpecifyModel(multires, clusters = 6:10, noise.pct = 0.1)
```

```R
modGMM <- clsGMM(gmmSpec, alpha = 0.2, beta = 0.2)
```
This took about 25 seconds on my computer.

![alt text](https://github.com/spoonbill/spoonbill.github.io/blob/master/images/clustersGMM1.png "Gaussian Mixture Model Clustering")
![alt text](https://github.com/spoonbill/spoonbill.github.io/blob/master/images/clustersGMM2.png "Gaussian Mixture Model Noise Filtering")

