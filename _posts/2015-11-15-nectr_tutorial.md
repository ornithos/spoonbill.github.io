---
layout: post
title: nectr: Exploratory Clustering
---
![_config.yml]({{ site.baseurl }}/images/config.png)
A number of years ago I spent some time researching clustering algorithms due to a growing dissatisfaction with k-means - the favourite clustering algorithm of hackers. During this time I was performing a lot of exploratory analysis, and I was hoping to understand the regions of high density within my dataset for a customer segmentation. Of course k-means cannot be trusted with such a task; while it may capture some elements of the underlying density, it is just as happy creating a relatively arbitrary partition. Further, if the true density is not given to spherical (or more accurately voronoi cell) clusters, then k-means will also fail as an accurate description of the data.

While non-parametric alternatives exist such as DBSCAN or mean-shift, these are highly sensitive to parameter specification, and higher dimensions suffer from quadratic time. Spectral clustering takes a slightly different approach by acting on the graph Laplacian, but unless approximated is at least quadratic, and one must specify the number of clusters in advance. The algorithm I chose is one called TURN-RES from a [paper](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.7.1966&rep=rep1&type=pdf) by Foss \& Za¨ıane in 2002. The main advantages are that it is a non parametric approach that is (sort-of) parameterless and runs in `O(nlogn)` time. It approximates density estimation by quantizing the space into discrete grid, assigning neighbours to each datapoint in all axis-oriented direction and calculating the number of neighbours within a given distance. For a few unclear details I have taken my best guess, but otherwise the implementation of TURN-RES follows the Foss paper.

## nectr: Non-Parametric Exploratory Clustering using TURN-RES
The R-package is called `nectr` and can be found [here](spoonbill/nectr). The following is a simple demonstration of the algorithm on some dummy data. First, we create a 5 cluster fake dataset over 3 dimensions according to the following R script:

```R
library(MASS)
n <- c(175, 60, 50, 135, 105) * 1000

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

fake <- as.data.frame(rbind(mvrnorm(n[1], mu[[1]], sigma[[1]]), mvrnorm(n[2], mu[[2]], sigma[[2]]),
                            mvrnorm(n[3], mu[[3]], sigma[[3]]), mvrnorm(n[4], mu[[4]], sigma[[4]]), 
                            mvrnorm(n[5],mu[[5]], sigma[[5]]), matrix(runif(n[6]*3,-10,10), n[6], 3)))
names(fake) <- c("X", "Y", "Z")
```
