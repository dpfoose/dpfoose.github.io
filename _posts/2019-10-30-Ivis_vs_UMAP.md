---
layout: post
title:  "UMAP vs. Ivis for Dimensionality Reduction"
date:   2019-10-29 12:35:22 -0400
categories: posts
---
Two promising dimensionality reduction algorithms were released last year, [Ivis](https://bering-ivis.readthedocs.io/en/latest/) and [UMAP](https://umap-learn.readthedocs.io/en/latest/).
The [Ivis paper](https://www.nature.com/articles/s41598-019-45301-0) was submitted to *Nature Scientific Reports* about 6 months after the [UMAP paper](https://arxiv.org/abs/1802.03426) appeared on arXiv,
and before UMAP came into general use. Both papers compare their new techniques favorably to [*t*-distributed Stochastic Neighbor Embedding (*t*-SNE)](http://www.jmlr.org/papers/v9/vandermaaten08a.html),
which has become one of the most popular nonlinear dimensionality reduction techniques. The Ivis creators have provided some benchmark data for both the 
[execution time](https://bering-ivis.readthedocs.io/en/latest/timings_benchmarks.html) and [distance preservation](https://bering-ivis.readthedocs.io/en/latest/embeddings_benchmarks.html) relative
to several other tehniques, including UMAP, *t*-SNE, [multidimensional scaling](https://en.wikipedia.org/wiki/Multidimensional_scaling) and 
[IsoMap](https://science.sciencemag.org/content/290/5500/2319), demonstrating that Ivis outperforms UMAP in terms of distance preservation and even execution time for some larger datasets. 

In this post, I will compare the performance of Ivis and UMAP using some of the examples used by the Ivis paper to compare
the performance of Ivis to *t*-SNE. For the datasets tested, Ivis is two orders of magnitude slower, but preserves global structure much better than UMAP.

The notebooks used to generate data and perform the dimensionality reductions is provided in [this repository]().
## Background
### UMAP
Uniform Manifold Approximation and Projection (UMAP) is a manifold learning technique based on [topological data analysis](https://en.wikipedia.org/wiki/Topological_data_analysis).
A fuzzy topological representation of the points is constructed in the higher dimensional space, then a lower-dimensional representation is optimized so that its fuzzy topological 
representation is as close as possible to that of the original representation, with fuzzy set membership calculated only for the *k* nearest neighbors of each point. 
#### Further Reading
* The [UMAP paper](https://arxiv.org/abs/1802.03426).
* [How UMAP Works](https://umap-learn.readthedocs.io/en/latest/how_umap_works.html)_ from the UMAP documentation.
* [How Exactly UMAP Works](https://towardsdatascience.com/how-exactly-umap-works-13e3040e1668) by Nikolay Oskolkov for Towards Data Science.

### Ivis
Ivis utilizes a [siamese network](https://en.wikipedia.org/wiki/Siamese_neural_network) with a modified form of the [triplet loss](https://en.wikipedia.org/wiki/Triplet_loss) function to create a lower-dimensional representation that 
minimizes the distance of each point to its nearest neighbors and maximizes the distance of each point to points which are not its nearest neighbors. 
#### Further Reading
* The [Ivis paper](https://www.nature.com/articles/s41598-019-45301-0).
* [Siamese Netowork & Triplet Loss](https://towardsdatascience.com/siamese-network-triplet-loss-b4ca82c1aec8) by Rohith Gandhi for Towards Data Science.

## Examples
For these examples, the 2D datasets are expanded into 9 dimensions using the same transformation as the Ivis paper: 
$$
(x, y) \rightarrow (x + y, x - y, xy, x^2, y^2, x^2y, xy^2, x^3, y^3)
$$.
To ease the comparison of the embeddings to the original 2D data, a [rigid transformation](https://gist.github.com/dpfoose/a22cfb9d19a8bb7a245086b51fc47a16) was applied to the embeddings 
to align them with the 2D data in a least squares sense. This can be done because the axes in both UMAP and Ivis are arbitrary.
Standardized the expanded data using `sklearn.preprocessing.StandardScaler` before fitting the dimensionality reduction did not improve either method, 
so the data presented is derived from unscaled expanded data.

I used the implementation of UMAP provided by [RAPIDS.ai's](https://rapids.ai/) [cuML](https://rapidsai.github.io/projects/cuml/en/0.9.0/api.html#umap) library and the Ivis implementation 
provided by the authors (`ivis` on PyPi).
### Random uniform distribution
We will start with the first example mentioned in the Ivis paper, a random sampling of 5000 points from the uniform distribution.
<div id="random-graph"></div>
Like *t*-SNE, UMAP creates spurious local structure, but the space is more "rectangular":
<div id="random-umap-graph"></div>
As in the Ivis paper, we see that Ivis does not create the spurious local structures created by *t*-SNE:
<div id="random-ivis-graph"></div>
### "Smiley" dataset
The next example from the Ivis paper is the "smiley" dataset, as generated by the [`mlbench` R package](https://rdrr.io/cran/mlbench/man/mlbench.smiley.html).
This data consists of 2 Gaussian eys, a trapezoid nose and a parabula mouth with vertical Gaussian noise. This example consists of 10000 points,
with the default standard deviations 0f 0.1 for the eyes and 0.05 for the mouth.
<div id="smiley-graph"></div>
UMAP maintains most local and some global structure, but misplaces the "nose" and the "mouth". 
<div id="smiley-umap-graph"></div>
Again, we see that Ivis does a decent job of preserving global structure.
<div id="smiley-ivis-graph"></div>

Using 0.35 as the standard deviation for both the eyes and the mouth creates more overlap between the classes:
<div id="loose-smiley-graph"></div>
The overlap means that the mouth is more likely to appear in the simplicial complexes of the eyes, which results in the relative positions of the eyes to the 
mouth being preserved. The nose is too far away to be included.
<div id="loose-smiley-umap-graph"></div>
Ivis does a decent job of maintaining global structure without as much distortion to the local structure.
<div id="loose-smiley-ivis-graph"></div>

### "Cassini problem" dataset
The "Cassini" dataset consists of points uniformly distributed within 3 structures, 2 external "bananas" and an inner circle:
<div id="cassini-graph"></div>
UMAP does not preserve the global structure at all, presenting the three structures as separate spheres:
<div id="cassini-umap-graph"></div>
Ivis does a pretty good job of preserving the overall global structure.
<div id="cassini-ivis-graph"></div>

### PBMC dataset
This example comes from [this R vignette](https://github.com/beringresearch/ivis/blob/master/R-package/vignettes/ivis_singlecell.Rmd), with the post-processing 
done using `scikit-learn` and the python `phenograph` package in place of the R package `Seurat`. 
The PBMC dataset was transformed using PCA, reducing its number of features to the number of samples. The data were then clustered using the [PhenoGraph](https://github.com/jacoblevine/PhenoGraph)
algorithm.
<div id="pbmc-graph"></div>
There are two outliers way outside of the range of the other points:
<div id="pbmc-umap-graph"></div>
Zooming in to exclude those outliers shows that UMAP performs about as well as Ivis:
<div id="pbmc-umap-graph-2"></div>
Ivis does about as good a job as UMAP and the first two principal components.
<div id="pbmc-ivis-graph"></div>

## Conclusions
Ivis outperforms UMAP at preserving global structure and should be considered for many applications of dimensionality reduction.

<script src="https://cdn.plot.ly/plotly-1.2.0.min.js"></script>
<script type="text/javascript" async src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.1/MathJax.js?..."></script>
<script>
  Plotly.d3.json('/assets/data/umap_ivis/random.json', (figure) => {
    figure.layout.title = 'Random Uniform Distribution';
    Plotly.newPlot('random-graph', figure.data, figure.layout);
  });
  Plotly.d3.json('/assets/data/umap_ivis/random_ivis.json', (figure) => {
    figure.layout.title = 'Random Uniform Distribution (Ivis Embeddings)';
    Plotly.newPlot('random-ivis-graph', figure.data, figure.layout)
  });
  Plotly.d3.json('/assets/data/umap_ivis/random_umap.json', (figure) => {
    figure.layout.title = 'Random Uniform Distribution (UMAP Embeddings)';
    Plotly.newPlot('random-umap-graph', figure.data, figure.layout);  
  });
  Plotly.d3.json('/assets/data/umap_ivis/smiley.json', (figure) => {
    figure.layout.title = '"Smiley"';
		Plotly.newPlot('smiley-graph', figure.data, figure.layout);
  });
  Plotly.d3.json('/assets/data/umap_ivis/smiley_ivis.json', (figure) => {
    figure.layout.title = '"Smiley" (Ivis Embeddings)';
    Plotly.newPlot('smiley-ivis-graph', figure.data, figure.layout)
  });
  Plotly.d3.json('/assets/data/umap_ivis/smiley_umap.json', (figure) => {
    figure.layout.title = '"Smiley" (UMAP Embeddings)';
    Plotly.newPlot('smiley-umap-graph', figure.data, figure.layout);  
  });
  Plotly.d3.json('/assets/data/umap_ivis/loose_smiley.json', (figure) => {
    figure.layout.title = 'Diffuse "Smiley"';
		Plotly.newPlot('loose-smiley-graph', figure.data, figure.layout);
  });
  Plotly.d3.json('/assets/data/umap_ivis/loose_smiley_ivis.json', (figure) => {
    figure.layout.title = 'Diffuse "Smiley" (Ivis Embeddings)';
		Plotly.newPlot('loose-smiley-ivis-graph', figure.data, figure.layout)
  });
  Plotly.d3.json('/assets/data/umap_ivis/loose_smiley_umap.json', (figure) => {
		figure.layout.title = 'Diffuse "Smiley" (UMAP Embeddings)';
    Plotly.newPlot('loose-smiley-umap-graph', figure.data, figure.layout);  
  });
  Plotly.d3.json('/assets/data/umap_ivis/cassini.json', (figure) => {
    figure.layout.title = 'Cassini';
    Plotly.newPlot('cassini-graph', figure.data, figure.layout);
  });
  Plotly.d3.json('/assets/data/umap_ivis/cassini_ivis.json', (figure) => {
    figure.layout.title = 'Cassini (Ivis Embeddings)';
    Plotly.newPlot('cassini-ivis-graph', figure.data, figure.layout)
  });
  Plotly.d3.json('/assets/data/umap_ivis/cassini_umap.json', (figure) => {
    figure.layout.title = 'Cassini (UMAP Embeddings)';
    Plotly.newPlot('cassini-umap-graph', figure.data, figure.layout);  
  });
  Plotly.d3.json('/assets/data/umap_ivis/pbmc.json', (figure) => {
    figure.layout.title='PBMC (PC2 vs PC1)';
    Plotly.newPlot('pbmc-graph', figure.data, figure.layout);
  });
  Plotly.d3.json('/assets/data/umap_ivis/pbmc_ivis.json', (figure) => {
    figure.layout.title = 'PBMC (Ivis Embeddings)';
    Plotly.newPlot('pbmc-ivis-graph', figure.data, figure.layout)
  });
  Plotly.d3.json('/assets/data/umap_ivis/pbmc_umap.json', (figure) => {
    let layout = figure.layout;
    layout.title = 'PBMC (UMAP Embeddings)';
    Plotly.newPlot('pbmc-umap-graph', figure.data, layout);  
  });
  Plotly.d3.json('/assets/data/umap_ivis/pbmc_umap.json', (figure) => {
    let layout = figure.layout;
    layout.title = 'PBMC (UMAP Embeddings)';
    console.log(layout);
    layout['xaxis'] = {range: [-2.6, 3]};
    layout['yaxis'] = {range: [-1.7, 0.7]};
    Plotly.newPlot('pbmc-umap-graph-2', figure.data, layout);  
  });
</script>

