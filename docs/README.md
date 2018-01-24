
<!-- README.md is generated from README.Rmd. Please edit that file -->
Overview
--------

The `MRFcov` package provides functions for approximating node interaction parameters of undirected Markov Random Fields (MRF) graphs. Models can incorporate covariates (a class of models known as [Conditional Markov Random Fields; CRFs](http://homepages.inf.ed.ac.uk/csutton/publications/crftut-fnt.pdf); following methods developed by Cheng *et al* 2014 and Lindberg 2016), allowing users to estimate how interactions between nodes in the graph are predicted to change across covariate gradients. In principle, these models are similar to joint species distribution models in that variance in species' occurrences can be partitioned into abiotic and biotic drivers. However, a key difference is that `MRFCov` models produce directly interpretable coefficients for determining the relative importances of these drivers. At present, only binary response variables can be included (1s and 0s), though models accomodating different data structures may be added in future.

MRF and CRF interaction parameters are approximated using separate regressions for individual species within a joint modelling framework. Because all combinations of covariates and additional species are included as predictor variables in node-specific regressions, variable selection is required to reduce overfitting and add sparsity. This is accomplished through LASSO penalization using functions in the [penalized](https://cran.r-project.org/web/packages/penalized/index.html) package.

*This project is licensed under the terms of the GNU General Public License (GNU GPLv3)*

Installation
------------

You can install the `MRFcov` package into `R` directly from `GitHub` using:

``` r
# install.packages("devtools")
devtools::install_github("nicholasjclark/MRFcov")
```

Usage
-----

We can explore the model's primary functions using a test dataset that is available with the package. Load the `Bird.parasites` dataset, which contains binary occurrences of four avian blood parasites in New Caledonian *Zosterops* species ([available in its original form at Dryad](http://dx.doi.org/10.5061/dryad.pp6k4); Clark *et al* 2016). A single continuous covariate is also included (`scale.prop.zos`), which reflects the relative abundance of *Zosterops* species among different sample sites

``` r
library(MRFcov)
data("Bird.parasites")
```

Visualise the dataset to see how analysis data needs to be structured. In short, node variable (i.e. species) occurrences should be included as binary variables (1s and 0s) as the left-most variables in `data`. Any covariates can be included as the right-most variables. Note, these covariates should all be on a similar scale, ideally using the `scale` function for continuous covariates (or similar) so that covariates have `mean = 0` and `sd = 1`

``` r
help("Bird.parasites")
View(Bird.parasites)
```

### Running MRFs and visualising interaction coefficients

Run an MRF model using the provided continuous covariate (`scale.prop.zos`). Here we specify a weak penalization parameter (`lambda1`) for the LASSO variable selection as we only have a single covariate

``` r
MRF_mod <- MRFcov(data = Bird.parasites, n_nodes = 4, lambda1 = 0.5)
```

Visualise the estimated species interaction coefficients as a heatmap

``` r
plotMRF_hm(MRF_mod = MRF_mod)
```

![](README-Readme.fig1-1.png)

Visualise how species interactions are predicted to change across covariate magnitudes

``` r
plotMRF_hm_cont(MRF_mod = MRF_mod, covariate = 'scale.prop.zos', data = Bird.parasites, 
                main = 'Estimated interactions across host relative densities')
```

![](README-Readme.fig2-1.png)

### Choosing penalization parameters

Choosing the appropriate `lambda1` value often requires exploration of model predictive performance at different values. The function `cv_MRF_diag` uses cross-validation to examine how well models fitted with training datasets can predict observations that have been with-held from the data (test datasets) at different `lambda1` values. We also have the option of specifying `lambda2`, where values `>0` lead to more effective shrinkage of coefficients (sometimes this is necessary for large datasets with many potential predictors; see documentation in the `penalized` package for more details). Given the large number of models that may be required during cross-validation or bootstrapping (see below), we can use parallel computation by relying on functions in the [parallel](https://www.google.com.au/url?sa=t&rct=j&q=&esrc=s&source=web&cd=2&cad=rja&uact=8&ved=0ahUKEwj6iZyPvcnYAhVLvrwKHaJ9AhUQFgg2MAE&url=https%3A%2F%2Fstat.ethz.ch%2FR-manual%2FR-devel%2Flibrary%2Fparallel%2Fdoc%2Fparallel.pdf&usg=AOvVaw2eR83aL93jttPIS-mLWzEL) package. The number of worker clusters to split the job across can be specified using the `n_cores` argument

``` r
cv_MRF_diag(data = Bird.parasites, min_lambda1 = 0.4, max_lambda1 = 2, by_lambda1 = 0.1, n_nodes = 4, n_cores = 3)
```

<img src="README-Readme.fig3-1.png" style="display: block; margin: auto;" />

    #> TableGrob (4 x 1) "arrange": 4 grobs
    #>   z     cells    name           grob
    #> 1 1 (1-1,1-1) arrange gtable[layout]
    #> 2 2 (2-2,1-1) arrange gtable[layout]
    #> 3 3 (3-3,1-1) arrange gtable[layout]
    #> 4 4 (4-4,1-1) arrange gtable[layout]

### Bootstrapping the data and running models across a range of penalization values

Here, `lambda1` values between 0.5 and 1.5 maintain reasonable **Sensitivity** (which represents the proportion of true positives that are correctly predicted). Because we are using rather rare parasite occurrences (prevalence of these parasites is fairly low), it is in our interest to use models that can maintain **Sensitivity**, as long as we are not at risk of overfitting (in this case, our maximum number of predictors is not very high, given that we only have four species and one covariate). Now that we have identified a suitable range of `lambda1` values, we can fit models to bootstrapped subsets of the data within this range to account for uncertainty.

``` r
booted_MRF <- bootstrap_MRF(data = Bird.parasites, n_nodes = 4, n_bootstraps = 50, min_lambda1 = 0.5, max_lambda1 = 1.5, by_lambda1 = 0.1, n_cores = 3)
```

Now we can visualise confidence intervals of interaction coefficients that were estimated across the full range of models

``` r
plotMRF_hm(MRF_mod = booted_MRF, plot_booted_coefs = TRUE)
```

<img src="README-Readme.fig4-1.png" style="display: block; margin: auto;" />

### Exploring regression coefficients and interpreting results

Finally, we can explore regression coefficients to get a better understanding of just how important interactions are for predicting species' occurrence probabilities (in comparison to other covariates). This is perhaps the strongest property of conditional MRFs, as competing methods (such as Joint Species Distribution Models) do not provide interpretable mechanisms for comparing the relative importances of interactions and fixed covariates. The `bootstrap_MRF` function conveniently returns a matrix of important coefficients for each node in the graph, as well as their relative importances (calculated using the formula `B^2 / sum(B^2)`, where the vector of `B`s represents regression coefficients for predictor variables). Variables with an underscore (`_`) indicate an interaction between a covariate and another node, suggesting that conditional dependencies of the two nodes vary across environmental gradients

``` r
booted_MRF$mean_key_coefs$Hzosteropis
#>                      Variable Rel_importance  Mean_coef
#> 1                  Hkillangoi     0.68253975 -3.3644904
#> 7 scale.prop.zos_Microfilaria     0.10582222 -1.3247800
#> 3                Microfilaria     0.06486330  1.0371817
#> 4              scale.prop.zos     0.06278876 -1.0204607
#> 6         scale.prop.zos_Plas     0.03619108  0.7747402
#> 2                        Plas     0.03339802 -0.7442446
#> 5   scale.prop.zos_Hkillangoi     0.01439688 -0.4886405
```

``` r
booted_MRF$mean_key_coefs$Hkillangoi
#>                     Variable Rel_importance  Mean_coef
#> 1                Hzosteropis     0.73132033 -3.3644904
#> 2               Microfilaria     0.09954285 -1.2412822
#> 5        scale.prop.zos_Plas     0.09920698  1.2391863
#> 3             scale.prop.zos     0.05124746 -0.8906389
#> 4 scale.prop.zos_Hzosteropis     0.01542581 -0.4886405
```

``` r
booted_MRF$mean_key_coefs$Plas
#>                      Variable Rel_importance  Mean_coef
#> 2                Microfilaria     0.45884569  1.9962474
#> 5   scale.prop.zos_Hkillangoi     0.17681174  1.2391863
#> 3              scale.prop.zos     0.15991089 -1.1784741
#> 4  scale.prop.zos_Hzosteropis     0.06911146  0.7747402
#> 6 scale.prop.zos_Microfilaria     0.06613015  0.7578457
#> 1                 Hzosteropis     0.06377776 -0.7442446
```

``` r
booted_MRF$mean_key_coefs$Microfilaria
#>                     Variable Rel_importance  Mean_coef
#> 3                       Plas     0.38836975  1.9962474
#> 5 scale.prop.zos_Hzosteropis     0.17104255 -1.3247800
#> 2                 Hkillangoi     0.15016118 -1.2412822
#> 4             scale.prop.zos     0.12928219 -1.1517574
#> 1                Hzosteropis     0.10483983  1.0371817
#> 6        scale.prop.zos_Plas     0.05597295  0.7578457
```

References
----------

Cheng, J., Levina, E., Wang, P. & Zhu, J. (2014). A sparse Ising model with covariates. *Biometrics* 70:943-953.

Clark, N.J., K. Wells, D. Dimitrov, and S.M. Clegg. (2016). Co-infections and environmental conditions drive the distributions of blood parasites in wild birds. *Journal of Animal Ecology* 85:1461-1470.

Lindberg, O. (2016). Markov Random Fields in Cancer Mutation Dependencies. Master's of Science Thesis. University of Turku, Turku, Finland.