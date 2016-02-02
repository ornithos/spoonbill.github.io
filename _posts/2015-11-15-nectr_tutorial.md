---
layout: post
title: nectr - Exploratory Clustering
---

A number of years ago I spent some time researching density-based clustering algorithms for exploratory analysis. At the time I was hoping to understand the regions of high density in a dataset for customer segmentation. Of course k-means cannot be trusted with such a task; it may capture some elements of the underlying density, but it is just as possible that it has created a relatively arbitrary partition of the data. Further, if the true density is not given to spherical (or more accurately voronoi cell) clusters, then k-means will also fail as an accurate description of the data. And then of course the perennial question of choosing K.

Parametric mixture models (most famously Gaussian Mixture Models) are more flexible with cluster shape, and confer various advantages against k-means, but I did not wish to assume any generating distribution. My experience of commercial data has not shown much evidence of any parametric distribution that I am familiar with. Various non-parametric alternatives exist such as DBSCAN or mean-shift, but these are highly sensitive to parameter specification, and higher dimensions suffer from quadratic time. Another approach is Spectral Clustering, using the graph Laplacian, but unless approximated is at least quadratic, and one must specify the number of clusters in advance. 

The algorithm I chose is one called TURN-RES from a [paper](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.7.1966&rep=rep1&type=pdf) by Foss \& Zaïane in 2002. The main advantages are that it is a non parametric approach that is (sort-of) parameterless and runs in `O(nlogn)` time. It approximates density estimation by quantizing the space into discrete grid, and computing the distance to all neighbours in axis-oriented directions. For a few unclear details I have taken my best guess, but otherwise the implementation of TURN-RES follows the Foss paper.

## TURN-RES
Two complementary algorithms are proposed in the Foss paper, TurnCut and TURN-RES. I believe TURN is an acronym, but I can't help you out with its meaning. The idea is that TURN-RES is the workhorse clustering algorithm, and TurnCut is its partner for choosing a good parameter. The latter proceeds in sequential manner calling TURN-RES and seeking to find the 'elbow' or 'knee' of various metrics (similar to those used for k-means). While the results for the experiments in the paper appear to be pretty good for TurnCut, I have not found it particularly useful in certain real world datasets. This is in part due to the subjectivity inherent in clustering; often a continuum of plausible results can be found for a particular parameter. But perhaps more importantly, one often finds that different clusters have different natural resolutions and a global resolution parameter is doomed to failure. 

This observation motivated a more hands-on approach, whereby instead of an algorithm attempting to find a solution to an ill-posed problem, the user is permitted to view a summary of intermediate results. By inspection, a suitable value or values of the resolution can be chosen. This process also can be a useful tool for a data scientist, in that understanding the density at various resolutions allows one to build up a more complete picture of the underlying distribution.

## Algorithmic overview

We'll quickly explore the algorithm via a poky animation. A window on some 2D data is shown below. The original data is shown in grey.
![dead](https://github.com/spoonbill/spoonbill.github.io/blob/master/images/clsTurnRes.gif "steps of TURN-RES")
+ Impose a grid at the given resolution (this is the only parameter choice in the algorithm).
+ Quantize the dataset -- move each datapoint to the nearest point on the grid.
+ For each datapoint, consider the neighbours in all axis-aligned directions.
+ If sufficient neighbours exist, the datapoint is labelled an 'interior' point and may be connected to each of its neighbours.
+ The union of all neighbouring interior points and their (exterior) neighbours form a cluster.
+ The data is returned to the original non-quantised space, together with the cluster labels.

The algorithm can be written to execute in `O(nlogn)` time, by partitioning the data into rows, sorting by column position, and obtaining horizontal neighbours. Vertical neighbours are discovered analogously. For higher dimensions, this process continues sequentially as sorts across each dimension. This may give you an insight into the effect of the curse of dimensionality on the algorithm: clearly this quantised approximation will perform poorly as the number of dimensions increase. It is only really recommended for `d < 10`.

## Parsimonious model compression using the GMM
While the non-parametric ideal is to be admired, there are many situations where it is simply impractical. Even mundane questions such as 'which cluster does my new datapoint belong to?' cannot be answered simply. It is even more difficult to quantify how central or otherwise a point is to its own cluster. For the customer segmentation case we also probably want to assign even outliers to their closest cluster. In order to help in these situations, a Gaussian Mixture Model has been implemented in the same package. It is integrated sufficiently well that the user need not know virtually anything about their construction. Because the exploratory work has already been performed, we know the number of clusters K, and the TURN clusters may be used as priors<sup>1</sup>. By selecting the strength of the prior, one can interpolate between the extremes of a mixture of Gaussians with mean and covariance of the TURN clusters, or simply an initialisation of the GMM from these cluster centers. Derivations of the MAP estimates are available in the repo.

## nectr: Non-Parametric Exploratory Clustering using TURN-RES

These functions are bundled together in the R-package `nectr` and can be found [here](https://github.com/spoonbill/nectr). The following is a simple demonstration of the algorithm on some dummy data. First, some fake data:

#### Create a 5 cluster fake dataset over 3 dimensions:

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

<div style="text-align:center"><img src="https://github.com/spoonbill/spoonbill.github.io/blob/master/images/clusters2.png" width="450"> <img src="https://github.com/spoonbill/spoonbill.github.io/blob/master/images/clusters1.png" width="450"></div>

#### Tuning the resolution parameter:
The core algorithm TURN-RES requires one parameter - the quantization resolution. The lower the resolution, the more neighbours a datapoint has, which increases the cluster sizes. As discussed above, searching for the optimal resolution is not automated, but the `clsMRes` function scans through some likely values in advance for you:

```R
library(nectr)
multires <- clsMRes(fakeData, keep = TRUE)
```
![alt text](https://github.com/spoonbill/spoonbill.github.io/blob/master/images/Rmultires.PNG "R output")
The clsMRes function iteratively reduces the resolution until no clusters are visible, and then increases until most datapoints are put in the same cluster. The `keep = TRUE` argument tells the function to keep all of the cluster assignments generated by the various iterations of TURN-RES. We may then use these to build a hierachical picture of how the clusters agglomerate. These 8 calls to the TURN-RES algorithm collectively took 30 seconds on a 2.7GHz i5 (single) processor.

#### Exploring the results
Inspection the agglomeration schedule, that is, how the clusters agglomerate as the scale is increased:
```R
plot(multires)
```

![alt text](https://github.com/spoonbill/spoonbill.github.io/blob/master/images/agglom.png "plot clsMR object")
The clusters are plotted in order of increasing resolution on the x axis. In this example, 5 clusters were found in the first run of TURN with resolution r = 0.102, numbered 1:5. Five (larger) clusters were discovered with the next resolution of r = 0.144, and the agglomeration schedule shows the clusters which are merged into a superset of themselves - for example cluster 6 is a superset of cluster 1. At the next iteration, clusters 6, 9 and 10 are merged into cluster 12. And so on.

Plotting selected clusters selected clusters of the agglomeration:
```R
plot(multires, c(8,12,13))
```
![alt text](https://github.com/spoonbill/spoonbill.github.io/blob/master/images/pcp8_12_13.png "Parallel Coordinate Plot")
This uses [Parallel Coordinate plots](https://en.wikipedia.org/wiki/Parallel_coordinates), where succesive dimensions are plotted on the x-axis. Note that clusters do not have to be of the same resolution to be plotted together, although sometimes this makes sense. Below we plot all the clusters at the highest level which retained 5 distinct clusters.

```R
plot(multires,6:10)
```
![alt text](https://github.com/spoonbill/spoonbill.github.io/blob/master/images/pcp_6_10.png "Parallel Coordinate Plot")
The parallel coordinate plots are a sample of the points belonging to each cluster, with the mean shown only implictly through these.

#### Creating the summary GMM model
The function `clsSpecifyModel` takes as arguments the multiresolution object created by `clsMRes`, the cluster numbers which we wish to keep in the GMM model (again these do not have to be from the same resolution level), and the estimated percentage of noise. The GMM will use the specified clusters as priors, and if a `noise.pct` is specified, an additional large variance noise cluster is admitted into the GMM to prevent outliers skewing the results<sup>2</sup>.
```R
gmmSpec <- clsSpecifyModel(multires, clusters = 6:10, noise.pct = 0.1)
```
Now we have specified the model, we must fit it. The following function takes the model as an argument, and `alpha`, `beta`, which are the strength of the priors used. These are scaled such that each prior strength should be specified in the interval `[0,1]`. `0` allows the GMM fit complete freedom in fitting, and the resulting model may look very different from the TURN model, at the other extreme, using `1` will simply return the covariance and means of the existing TURN clusters. Any value in the middle is a trade off between these objectives, and below we have used `0.2`. Different values may be specified for each cluster if desired.
```R
modGMM <- clsGMM(gmmSpec, alpha = 0.2, beta = 0.2)
```
The model is fitted by standard EM, which took about 25 seconds for this example on my computer. The results are shown below; both the 5 fitted clusters, discovered using TURN, and described using the GMM, and in the second, the effect of filtering out low density observations as noise.

<div style="text-align:center"><img src="https://github.com/spoonbill/spoonbill.github.io/blob/master/images/clustersGMM1.png" alt = "Gaussian Mixture Model Clustering" width="550"> <img src="https://github.com/spoonbill/spoonbill.github.io/blob/master/images/clustersGMM2.png" alt = Gaussian Mixture Model Noise Filtering" width="550"></div>


<sup>1</sup> the GMM uses the MAP not MLE estimate.

<sup>2</sup> Gaussian distributions are notoriously sensitive to outliers, due to the double exponential decay of the density.