---
header:
  caption: ""
  image: ""
title: Random pieces of code
view: 1
---

Below I provide pieces of code I have written over the years and shared through [GitHub](https://github.com/oliviergimenez) on [reproducible science](#reproducible-science), [capture-recapture](#capture-recapture),  [occupancy](#occupancy) and [epidemiological](#epidemiological-models-and-odes) models, [spatial stuff](#spatial), [social networks](#social-networks), [deep learning](#deep-learning-and-species-identification), [textual analyses](#bibliometric-and-textual-analyses), the [tidyverse](#tidyverse) and some [random stuff](#random-stuff).

This code is provided as is without warranty of any kind. If you find a bug or anything, feel free to [get in touch](mailto:olivier.gimenez@cefe.cnrs.fr) or [fill an issue on GitHub](https://docs.github.com/en/github/managing-your-work-on-github/creating-an-issue). 

You may find other pieces of code in [papers](https://oliviergimenez.github.io/publication/papers/).

<hr>

### Reproducible science

<br>

+ Introduction to `Git` and `GitHub` with `RStudio` [[slides](https://oliviergimenez.github.io/quick-intro-git-github-rstudio/#1)] [[material](https://github.com/oliviergimenez/quick-intro-git-github-rstudio)]

+ Writing dynamic and reproducible documents with `R Markdown` [[slides](https://oliviergimenez.github.io/intro_rmarkdown/#1)] [[material](https://github.com/oliviergimenez/intro_rmarkdown)] [[practical](https://github.com/oliviergimenez/intro_rmarkdown_practical)] [[interactive dashboard](https://github.com/oliviergimenez/bias_occupancy)] [[website](https://www.youtube.com/watch?v=4OUEss2XF7E&t=1s)]

+ Manipulate and visualise data in the `tidyverse` [[slides](https://oliviergimenez.github.io/intro_tidyverse/#1)] [[material](https://github.com/oliviergimenez/intro_tidyverse)] [[tips](https://oliviergimenez.github.io/tidyverse-tips/)]

+ Focus on GIS and spatial data with `sf` package [[slides](https://oliviergimenez.github.io/intro_spatialR/#1)] [[material](https://github.com/oliviergimenez/intro_spatialR)]

<hr>

### Capture-recapture

<br>

+ [Capture-recapture models as HMMs](https://github.com/oliviergimenez/IPMworkshop) -- `R`, `Jags` 

+ [Spatial capture-recapture models](https://github.com/oliviergimenez/basics_spatial_capturerecapture) for closed and open populations -- `R`, `Nimble` 

+ [Jolly-Seber](https://github.com/oliviergimenez/bayes-multistate-jollyseber) capture-recapture models -- `R`, `Jags`

+ [Multievent](https://github.com/oliviergimenez/multievent_jags_R) capture-recapture models --  `R`, `Jags`, `Nimble`

+ [Multievent](https://github.com/oliviergimenez/multieventRcpp) capture-recapture models with Rcpp --  `R`, `Rcpp`

+ [Multievent SIR models](https://github.com/oliviergimenez/sir_multievent) and capture-recapture data --  `R`, `TMB`

+ [Local minima](https://github.com/oliviergimenez/multistate_local_minima) and multistate capture-recapture models; see also [there](https://github.com/oliviergimenez/multistate_recaprecov) for mixture of recaptures and recoveries --  `R`, `Jags`, `ADMB`, `Stan`, `unmarked`

+ [Animal social networks](https://github.com/oliviergimenez/social_networks_capture_recapture) and capture-recapture data --  `R`, `Jags`
   
+ [Individual fitness in the wild](https://github.com/oliviergimenez/estim_fitness) and capture-recapture data --  `R`, `Jags`, `E-SURGE`

+ Gompertz capture-recapture model with a [Gamma frailty](https://github.com/oliviergimenez/GammaGompertzCR) --  `R`, `Jags`

+ [Many correlated covariates](https://github.com/oliviergimenez/p2cr) in capture-recapture models --  `R`, `RMark`

+  [Structural equation](https://github.com/oliviergimenez/sem_capturerecapture) capture-recapture models --  `R`, `Jags`

+ [Abundance estimation](https://github.com/oliviergimenez/abundance_estimation) using capture-recapture models --  `R`, `RMark`

+ [Band-recovery models with time random effects](https://github.com/oliviergimenez/mixed-bandrecovery-models) in a frequentist framework  --  `R`

+ Bayesian implementation of capture-recapture models with [robust design](https://github.com/oliviergimenez/bayesRD)  --  `R`, `Jags`

+ Predator-prey [integrated community models](https://github.com/oliviergimenez/predator_prey_icm) --  `R`, `Jags`

<hr>

### Occupancy

<br>

+ [Dynamic analysis](https://github.com/oliviergimenez/ursus_Pyrenees_occupancy) of brown bear habitat use in the Pyrenees --  `R`, `unmarked`

+ Dynamic occupancy models in [`TMB`](https://github.com/oliviergimenez/occupancy_TMB) and [`ADMB`](https://github.com/oliviergimenez/occupancy_in_ADMB) -- `R`, `TMB`, `ADMB`

+ [Multistate occupancy model with uncertainty](https://github.com/oliviergimenez/multistate_occupancy)  --  `R`, `Jags`

+ [HMM formulation of a multispecies dynamic](https://github.com/oliviergimenez/poaching_occupancy) occupancy models --  `R`

+ Simulation and fit of a 2-species occupancy model à la Rota et al. (2016) [here](https://github.com/oliviergimenez/2speciesoccupancy) with `unmarked` and [there](https://github.com/oliviergimenez/bayes2speciesoccupancy) with `Jags` and `Nimble`

+ [Bias in occupancy](https://github.com/oliviergimenez/bias_occupancy) estimate for a static occupancy model, with an [interactive dashboard](https://ecologicalstatistics.shinyapps.io/bias_occupancy/) -- `R` and `unmarked`

<hr>

### Spatial

<br>


+ [Animated map](https://github.com/oliviergimenez/tidytuesday/tree/master/2019/week52) of wolf presence in France --  `R`

+ Distribution map from an occupancy model, with [Eurasian](https://github.com/oliviergimenez/gis_map_occupancy) or [Balkan](https://github.com/oliviergimenez/occ_balkanlynx) lynx --  `R`, `unmarked`
   
+ Introduction to [GIS and mapping in `R`](https://github.com/oliviergimenez/intro_spatialR) using the `sf` package with [slides](https://oliviergimenez.github.io/introspatialR/) --  `R`

<hr>

### Epidemiological models and ODEs

<br>


+ Dynamic models based on ODEs [here](https://github.com/oliviergimenez/fitODEnimble) and [there](https://github.com/oliviergimenez/ODEnimble) --  `Nimble` 

+  Explore strategies of [social distancing on epidemic w/ SIR model](https://github.com/oliviergimenez/SIRcovid19#motivation) -- `R`

+ [Zombie apocalypse](https://github.com/oliviergimenez/fitODEswithOpenBUGS) and a Bayesian approach for fitting an epidemiological model to data --  `R`, `OpenBUGS`

<hr>

### Social networks

<br>


+ Network analysis of [Star Wars The Attack of the Clones](https://github.com/oliviergimenez/starwars_network) in R
   
+ Animal social networks from [capture-recapture data](https://github.com/oliviergimenez/social_networks_capture_recapture)

+ Analysis of [Bottlenose dolphin](https://github.com/oliviergimenez/tursiops_network_analysis) social network --  `R`

+ [Scientific research](https://github.com/oliviergimenez/phd-in-ecology-network) is all about networking --  `R`

<hr>

### Deep learning and species identification

<br>


+ Piégeage photo et pipeline pour l'identification d'espèces via [deep learning](https://github.com/oliviergimenez/DLcamtrap#pi%C3%A9geage-photo-et-pipeline-pour-lidentification-desp%C3%A8ces) -- `Python`, `R`

<hr> 

### Bibliometric and textual analyses

<br>


+ [Content analysis](https://github.com/oliviergimenez/wolf_content_analysis) of French articles in newspapers about wolf  --  `R`

+ Bibliometric and textual analyses of the [last decade of research in capture-recapture](https://github.com/oliviergimenez/appendix_capturerecapture_review) --  `R`

+ Quick and dirty [analyses of ISEC 2018 tweets](https://github.com/oliviergimenez/isec2018_tweet_analysis) --  `R`

+ Text mining of the [ISEC 2020 abstracts](https://github.com/oliviergimenez/textmining_isec2020/blob/master/textmining_isec2020abstracts.R) --  `R`


<hr>

### Tidyverse

<br>


+ [Introduction](https://github.com/oliviergimenez/intro_tidyverse#introduction-to-the-tidyverse) to the tidyverse --  `R`

+ Some notes I have taken on David Robinson's screencasts, with [`R` tidyverse tips and tricks](https://oliviergimenez.github.io/tidyverse-tips/) --  `R`

+ Analyses of [wolf depredation in France](https://github.com/oliviergimenez/analysesGeoloup) --  `R`

+ Visualizing [trends in French deaths](https://github.com/oliviergimenez/deces_coulmont) --  `R` 


<hr>

### Random stuff

<br>


+ Illustration of the [delta-method](https://github.com/oliviergimenez/delta_method) --  `R`

+ Creating a [hex sticker](https://github.com/oliviergimenez/R2ucarehex) for the  `R2ucare` package --  `R`

+ [Simulate data with `Jags`](https://github.com/oliviergimenez/simul_with_jags)  --  `R`, `Jags`

+ [First steps in Python](https://github.com/oliviergimenez/firststepsinPython) -- `Python`

+ [Bayesian Multivariate Autoregressive State-Space](https://github.com/oliviergimenez/marss_jags) models - MARSS --  `R`, `Jags`



