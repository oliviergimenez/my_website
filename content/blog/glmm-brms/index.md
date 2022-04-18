+++
date = "2022-04-18"
draft = false
tags = ["GLMM", "bayes", "R", "brms", "jags"]
title = "Bayesian analyses made easy: GLMMs in R package brms"
math = true
summary = """
"""

+++

Here I illustrate how to fit GLMMs with the R package `brms`, and compare to `Jags` and `lme4`. 

<!--more-->

# Motivation

I regularly give [a course on Bayesian statistics with `R` for non-specialists](https://oliviergimenez.github.io/bayesian-stats-with-R/). To illustrate the course, we analyse data with [generalized linear, often mixed, models or GLMMs](https://bbolker.github.io/mixedmodels-misc/glmmFAQ.html). 

So far, I've been using `Jags` to fit these models. This requires some programming skills, like e.g. coding a loop, to be able to write down the model likelihood. Although students learn a lot from going through that process, it can be daunting. 

This year, I thought I'd show them the [`R` package `brms`](https://paul-buerkner.github.io/brms/) developed by [Paul-Christian Bürkner](https://paul-buerkner.github.io/). In brief, `brms` allows fitting GLMMs (but not only) in a `lme4`-like syntax within the Bayesian framework and MCMC methods with Stan.  

I'm not a Stan user, but it doesn't matter. The [vignettes](https://paul-buerkner.github.io/brms/articles/) were more than enough to get me started. I also recommend the [list of blog posts about `brms`](https://paul-buerkner.github.io/blog/brms-blogposts/). 

First things first, we load the packages we will need:

```r
library(brms)
library(posterior) # tools for working with posterior and prior distributions
library(R2jags) # run Jags from within R
library(lme4) # fit GLMM in frequentist framework
```

# Beta-Binomial model

The first example is about a survival experiment. We captured, marked and released 57 individuals (*total*) out of which 19 made it through the year (*alive*). 

```r
dat <- data.frame(alive = 19, total = 57)
dat
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":["alive"],"name":[1],"type":["dbl"],"align":["right"]},{"label":["total"],"name":[2],"type":["dbl"],"align":["right"]}],"data":[{"1":"19","2":"57"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>

An obvious estimate of the probability of a binomial is the proportion of cases, $19/57 = 0.3333333$ here, but let's use `R` built-in functions for the sake of illustration.

## Maximum-likelihood estimation

To get an estimate of survival, we fit a logistic regression using the `glm()` function. The data are grouped (or aggregated), so we need to specify the number of alive and dead individuals as a two-column matrix on the left hand side of the model formula:

```r
mle <- glm(cbind(alive, total - alive) ~ 1, 
           family = binomial("logit"), # family = binomial("identity") would be more straightforward
           data = dat)
```

On the logit scale, survival is estimated at:

```r
coef(mle) # logit scale
```

```
## (Intercept) 
##  -0.6931472
```

After back-transformation using the reciprocal logit, we obtain:

```r
plogis(coef(mle))
```

```
## (Intercept) 
##   0.3333333
```

So far, so good. 

## Bayesian analysis with `Jags`

In `Jags`, you can fit this model with:

```r
# model
betabin <- function(){
  alive ~ dbin(survival, total) # binomial likelihood
  logit(survival) <- beta # logit link
  beta ~ dnorm(0, 1/1.5) # prior
}
# data
datax <- list(total = 57,
              alive = 19)
# initial values
inits <- function() list(beta = rnorm(1, 0, sd = 1.5))
# parameter to monitor
params <- c("survival")
# run jags
bayes.jags <- jags(data = datax,
                      inits = inits,
                      parameters.to.save = params,
                      model.file = betabin,
                      n.chains = 2,
                      n.iter = 5000,
                      n.burnin = 1000,
                      n.thin = 1)
```

```
## Compiling model graph
##    Resolving undeclared variables
##    Allocating nodes
## Graph information:
##    Observed stochastic nodes: 1
##    Unobserved stochastic nodes: 1
##    Total graph size: 8
## 
## Initializing model
```

```r
# display results
bayes.jags
```

```
## Inference for Bugs model at "/var/folders/r7/j0wqj1k95vz8w44sdxzm986c0000gn/T//RtmpP7MZ1A/modela81d72dd23e2.txt", fit using jags,
##  2 chains, each with 5000 iterations (first 1000 discarded)
##  n.sims = 8000 iterations saved
##          mu.vect sd.vect  2.5%   25%   50%   75% 97.5%  Rhat n.eff
## survival   0.342   0.061 0.228 0.300 0.341 0.382 0.466 1.001  2600
## deviance   5.354   1.390 4.388 4.486 4.819 5.662 9.333 1.001  8000
## 
## For each parameter, n.eff is a crude measure of effective sample size,
## and Rhat is the potential scale reduction factor (at convergence, Rhat=1).
## 
## DIC info (using the rule, pD = var(deviance)/2)
## pD = 1.0 and DIC = 6.3
## DIC is an estimate of expected predictive error (lower deviance is better).
```

## Bayesian analysis with `brms`

In `brms`, you write:

```r
bayes.brms <- brm(alive | trials(total) ~ 1, 
                  family = binomial("logit"), # binomial("identity") would be more straightforward
                  data = dat,
                  chains = 2, # nb of chains
                  iter = 5000, # nb of iterations, including burnin
                  warmup = 1000, # burnin
                  thin = 1) # thinning
```

```
## Running /Library/Frameworks/R.framework/Resources/bin/R CMD SHLIB foo.c
## clang -mmacosx-version-min=10.13 -I"/Library/Frameworks/R.framework/Resources/include" -DNDEBUG   -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/Rcpp/include/"  -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/"  -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/unsupported"  -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/BH/include" -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/StanHeaders/include/src/"  -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/StanHeaders/include/"  -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppParallel/include/"  -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/rstan/include" -DEIGEN_NO_DEBUG  -DBOOST_DISABLE_ASSERTS  -DBOOST_PENDING_INTEGER_LOG2_HPP  -DSTAN_THREADS  -DBOOST_NO_AUTO_PTR  -include '/Library/Frameworks/R.framework/Versions/4.1/Resources/library/StanHeaders/include/stan/math/prim/mat/fun/Eigen.hpp'  -D_REENTRANT -DRCPP_PARALLEL_USE_TBB=1   -I/usr/local/include   -fPIC  -Wall -g -O2  -c foo.c -o foo.o
## In file included from <built-in>:1:
## In file included from /Library/Frameworks/R.framework/Versions/4.1/Resources/library/StanHeaders/include/stan/math/prim/mat/fun/Eigen.hpp:13:
## In file included from /Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/Eigen/Dense:1:
## In file included from /Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/Eigen/Core:88:
## /Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/Eigen/src/Core/util/Macros.h:628:1: error: unknown type name 'namespace'
## namespace Eigen {
## ^
## /Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/Eigen/src/Core/util/Macros.h:628:16: error: expected ';' after top level declarator
## namespace Eigen {
##                ^
##                ;
## In file included from <built-in>:1:
## In file included from /Library/Frameworks/R.framework/Versions/4.1/Resources/library/StanHeaders/include/stan/math/prim/mat/fun/Eigen.hpp:13:
## In file included from /Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/Eigen/Dense:1:
## /Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/Eigen/Core:96:10: fatal error: 'complex' file not found
## #include <complex>
##          ^~~~~~~~~
## 3 errors generated.
## make: *** [foo.o] Error 1
## 
## SAMPLING FOR MODEL '782790eabbd10296f779d85fa03cf1e5' NOW (CHAIN 1).
## Chain 1: 
## Chain 1: Gradient evaluation took 2e-05 seconds
## Chain 1: 1000 transitions using 10 leapfrog steps per transition would take 0.2 seconds.
## Chain 1: Adjust your expectations accordingly!
## Chain 1: 
## Chain 1: 
## Chain 1: Iteration:    1 / 5000 [  0%]  (Warmup)
## Chain 1: Iteration:  500 / 5000 [ 10%]  (Warmup)
## Chain 1: Iteration: 1000 / 5000 [ 20%]  (Warmup)
## Chain 1: Iteration: 1001 / 5000 [ 20%]  (Sampling)
## Chain 1: Iteration: 1500 / 5000 [ 30%]  (Sampling)
## Chain 1: Iteration: 2000 / 5000 [ 40%]  (Sampling)
## Chain 1: Iteration: 2500 / 5000 [ 50%]  (Sampling)
## Chain 1: Iteration: 3000 / 5000 [ 60%]  (Sampling)
## Chain 1: Iteration: 3500 / 5000 [ 70%]  (Sampling)
## Chain 1: Iteration: 4000 / 5000 [ 80%]  (Sampling)
## Chain 1: Iteration: 4500 / 5000 [ 90%]  (Sampling)
## Chain 1: Iteration: 5000 / 5000 [100%]  (Sampling)
## Chain 1: 
## Chain 1:  Elapsed Time: 0.010358 seconds (Warm-up)
## Chain 1:                0.046464 seconds (Sampling)
## Chain 1:                0.056822 seconds (Total)
## Chain 1: 
## 
## SAMPLING FOR MODEL '782790eabbd10296f779d85fa03cf1e5' NOW (CHAIN 2).
## Chain 2: 
## Chain 2: Gradient evaluation took 1.3e-05 seconds
## Chain 2: 1000 transitions using 10 leapfrog steps per transition would take 0.13 seconds.
## Chain 2: Adjust your expectations accordingly!
## Chain 2: 
## Chain 2: 
## Chain 2: Iteration:    1 / 5000 [  0%]  (Warmup)
## Chain 2: Iteration:  500 / 5000 [ 10%]  (Warmup)
## Chain 2: Iteration: 1000 / 5000 [ 20%]  (Warmup)
## Chain 2: Iteration: 1001 / 5000 [ 20%]  (Sampling)
## Chain 2: Iteration: 1500 / 5000 [ 30%]  (Sampling)
## Chain 2: Iteration: 2000 / 5000 [ 40%]  (Sampling)
## Chain 2: Iteration: 2500 / 5000 [ 50%]  (Sampling)
## Chain 2: Iteration: 3000 / 5000 [ 60%]  (Sampling)
## Chain 2: Iteration: 3500 / 5000 [ 70%]  (Sampling)
## Chain 2: Iteration: 4000 / 5000 [ 80%]  (Sampling)
## Chain 2: Iteration: 4500 / 5000 [ 90%]  (Sampling)
## Chain 2: Iteration: 5000 / 5000 [100%]  (Sampling)
## Chain 2: 
## Chain 2:  Elapsed Time: 0.009763 seconds (Warm-up)
## Chain 2:                0.035949 seconds (Sampling)
## Chain 2:                0.045712 seconds (Total)
## Chain 2:
```

You can display the results:

```r
bayes.brms
```

```
##  Family: binomial 
##   Links: mu = logit 
## Formula: alive | trials(total) ~ 1 
##    Data: dat (Number of observations: 1) 
##   Draws: 2 chains, each with iter = 5000; warmup = 1000; thin = 1;
##          total post-warmup draws = 8000
## 
## Population-Level Effects: 
##           Estimate Est.Error l-95% CI u-95% CI Rhat Bulk_ESS Tail_ESS
## Intercept    -0.69      0.28    -1.27    -0.14 1.00     3213     3572
## 
## Draws were sampled using sampling(NUTS). For each parameter, Bulk_ESS
## and Tail_ESS are effective sample size measures, and Rhat is the potential
## scale reduction factor on split chains (at convergence, Rhat = 1).
```

And visualize the posterior density and trace of survival (on the logit scale):

```r
plot(bayes.brms)
```

![](unnamed-chunk-9-1.png)<!-- -->

To get survival on the $[0,1]$ scale, we extract the MCMC values, then apply the reciprocal logit function to each of these values, and summarize its posterior distribution: 

```r
draws_fit <- as_draws_matrix(bayes.brms)
draws_fit
```

```
## # A draws_matrix: 4000 iterations, 2 chains, and 2 variables
##     variable
## draw b_Intercept lp__
##   1        -0.70 -4.2
##   2        -0.67 -4.2
##   3        -0.86 -4.4
##   4        -0.80 -4.3
##   5        -1.32 -6.6
##   6        -0.57 -4.2
##   7        -0.51 -4.4
##   8        -0.56 -4.3
##   9        -0.69 -4.2
##   10       -0.41 -4.6
## # ... with 7990 more draws
```

```r
summarize_draws(plogis(draws_fit[,1]))
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":["variable"],"name":[1],"type":["chr"],"align":["left"]},{"label":["mean"],"name":[2],"type":["dbl"],"align":["right"]},{"label":["median"],"name":[3],"type":["dbl"],"align":["right"]},{"label":["sd"],"name":[4],"type":["dbl"],"align":["right"]},{"label":["mad"],"name":[5],"type":["dbl"],"align":["right"]},{"label":["q5"],"name":[6],"type":["dbl"],"align":["right"]},{"label":["q95"],"name":[7],"type":["dbl"],"align":["right"]},{"label":["rhat"],"name":[8],"type":["dbl"],"align":["right"]},{"label":["ess_bulk"],"name":[9],"type":["dbl"],"align":["right"]},{"label":["ess_tail"],"name":[10],"type":["dbl"],"align":["right"]}],"data":[{"1":"b_Intercept","2":"0.3372479","3":"0.3346894","4":"0.06186131","5":"0.06164302","6":"0.2388592","7":"0.4422435","8":"1.001294","9":"3212.932","10":"3571.757"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>

What is the prior used by default? 

```r
prior_summary(bayes.brms)
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":["prior"],"name":[1],"type":["chr"],"align":["left"]},{"label":["class"],"name":[2],"type":["chr"],"align":["left"]},{"label":["coef"],"name":[3],"type":["chr"],"align":["left"]},{"label":["group"],"name":[4],"type":["chr"],"align":["left"]},{"label":["resp"],"name":[5],"type":["chr"],"align":["left"]},{"label":["dpar"],"name":[6],"type":["chr"],"align":["left"]},{"label":["nlpar"],"name":[7],"type":["chr"],"align":["left"]},{"label":["bound"],"name":[8],"type":["chr"],"align":["left"]},{"label":["source"],"name":[9],"type":["chr"],"align":["left"]}],"data":[{"1":"student_t(3, 0, 2.5)","2":"Intercept","3":"","4":"","5":"","6":"","7":"","8":"","9":"default"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>

What if I want to use the same prior as in `Jags` instead? 

```r
nlprior <- prior(normal(0, 1.5), class = "Intercept") # new prior
bayes.brms <- brm(alive | trials(total) ~ 1, 
                  family = binomial("logit"), 
                  data = dat, 
                  prior = nlprior, # set prior by hand
                  chains = 2, # nb of chains
                  iter = 5000, # nb of iterations, including burnin
                  warmup = 1000, # burnin
                  thin = 1)
```

```
## Running /Library/Frameworks/R.framework/Resources/bin/R CMD SHLIB foo.c
## clang -mmacosx-version-min=10.13 -I"/Library/Frameworks/R.framework/Resources/include" -DNDEBUG   -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/Rcpp/include/"  -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/"  -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/unsupported"  -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/BH/include" -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/StanHeaders/include/src/"  -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/StanHeaders/include/"  -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppParallel/include/"  -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/rstan/include" -DEIGEN_NO_DEBUG  -DBOOST_DISABLE_ASSERTS  -DBOOST_PENDING_INTEGER_LOG2_HPP  -DSTAN_THREADS  -DBOOST_NO_AUTO_PTR  -include '/Library/Frameworks/R.framework/Versions/4.1/Resources/library/StanHeaders/include/stan/math/prim/mat/fun/Eigen.hpp'  -D_REENTRANT -DRCPP_PARALLEL_USE_TBB=1   -I/usr/local/include   -fPIC  -Wall -g -O2  -c foo.c -o foo.o
## In file included from <built-in>:1:
## In file included from /Library/Frameworks/R.framework/Versions/4.1/Resources/library/StanHeaders/include/stan/math/prim/mat/fun/Eigen.hpp:13:
## In file included from /Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/Eigen/Dense:1:
## In file included from /Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/Eigen/Core:88:
## /Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/Eigen/src/Core/util/Macros.h:628:1: error: unknown type name 'namespace'
## namespace Eigen {
## ^
## /Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/Eigen/src/Core/util/Macros.h:628:16: error: expected ';' after top level declarator
## namespace Eigen {
##                ^
##                ;
## In file included from <built-in>:1:
## In file included from /Library/Frameworks/R.framework/Versions/4.1/Resources/library/StanHeaders/include/stan/math/prim/mat/fun/Eigen.hpp:13:
## In file included from /Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/Eigen/Dense:1:
## /Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/Eigen/Core:96:10: fatal error: 'complex' file not found
## #include <complex>
##          ^~~~~~~~~
## 3 errors generated.
## make: *** [foo.o] Error 1
## 
## SAMPLING FOR MODEL '59dcc499a2b4c8a1b2c8f4e80e70fcce' NOW (CHAIN 1).
## Chain 1: 
## Chain 1: Gradient evaluation took 1.8e-05 seconds
## Chain 1: 1000 transitions using 10 leapfrog steps per transition would take 0.18 seconds.
## Chain 1: Adjust your expectations accordingly!
## Chain 1: 
## Chain 1: 
## Chain 1: Iteration:    1 / 5000 [  0%]  (Warmup)
## Chain 1: Iteration:  500 / 5000 [ 10%]  (Warmup)
## Chain 1: Iteration: 1000 / 5000 [ 20%]  (Warmup)
## Chain 1: Iteration: 1001 / 5000 [ 20%]  (Sampling)
## Chain 1: Iteration: 1500 / 5000 [ 30%]  (Sampling)
## Chain 1: Iteration: 2000 / 5000 [ 40%]  (Sampling)
## Chain 1: Iteration: 2500 / 5000 [ 50%]  (Sampling)
## Chain 1: Iteration: 3000 / 5000 [ 60%]  (Sampling)
## Chain 1: Iteration: 3500 / 5000 [ 70%]  (Sampling)
## Chain 1: Iteration: 4000 / 5000 [ 80%]  (Sampling)
## Chain 1: Iteration: 4500 / 5000 [ 90%]  (Sampling)
## Chain 1: Iteration: 5000 / 5000 [100%]  (Sampling)
## Chain 1: 
## Chain 1:  Elapsed Time: 0.00875 seconds (Warm-up)
## Chain 1:                0.03345 seconds (Sampling)
## Chain 1:                0.0422 seconds (Total)
## Chain 1: 
## 
## SAMPLING FOR MODEL '59dcc499a2b4c8a1b2c8f4e80e70fcce' NOW (CHAIN 2).
## Chain 2: 
## Chain 2: Gradient evaluation took 5e-06 seconds
## Chain 2: 1000 transitions using 10 leapfrog steps per transition would take 0.05 seconds.
## Chain 2: Adjust your expectations accordingly!
## Chain 2: 
## Chain 2: 
## Chain 2: Iteration:    1 / 5000 [  0%]  (Warmup)
## Chain 2: Iteration:  500 / 5000 [ 10%]  (Warmup)
## Chain 2: Iteration: 1000 / 5000 [ 20%]  (Warmup)
## Chain 2: Iteration: 1001 / 5000 [ 20%]  (Sampling)
## Chain 2: Iteration: 1500 / 5000 [ 30%]  (Sampling)
## Chain 2: Iteration: 2000 / 5000 [ 40%]  (Sampling)
## Chain 2: Iteration: 2500 / 5000 [ 50%]  (Sampling)
## Chain 2: Iteration: 3000 / 5000 [ 60%]  (Sampling)
## Chain 2: Iteration: 3500 / 5000 [ 70%]  (Sampling)
## Chain 2: Iteration: 4000 / 5000 [ 80%]  (Sampling)
## Chain 2: Iteration: 4500 / 5000 [ 90%]  (Sampling)
## Chain 2: Iteration: 5000 / 5000 [100%]  (Sampling)
## Chain 2: 
## Chain 2:  Elapsed Time: 0.009788 seconds (Warm-up)
## Chain 2:                0.036797 seconds (Sampling)
## Chain 2:                0.046585 seconds (Total)
## Chain 2:
```

Double-check the prior that was used:

```r
prior_summary(bayes.brms)
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":["prior"],"name":[1],"type":["chr"],"align":["left"]},{"label":["class"],"name":[2],"type":["chr"],"align":["left"]},{"label":["coef"],"name":[3],"type":["chr"],"align":["left"]},{"label":["group"],"name":[4],"type":["chr"],"align":["left"]},{"label":["resp"],"name":[5],"type":["chr"],"align":["left"]},{"label":["dpar"],"name":[6],"type":["chr"],"align":["left"]},{"label":["nlpar"],"name":[7],"type":["chr"],"align":["left"]},{"label":["bound"],"name":[8],"type":["chr"],"align":["left"]},{"label":["source"],"name":[9],"type":["chr"],"align":["left"]}],"data":[{"1":"normal(0, 1.5)","2":"Intercept","3":"","4":"","5":"","6":"","7":"","8":"","9":"user"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>

# Logistic regression with covariates

In this example, we ask whether annual variation in white stork breeding success can be explained by rainfall in the wintering area. Breeding success is measured by the ratio of the number of chicks over the number of pairs. 

```r
nbchicks <- c(151,105,73,107,113,87,77,108,118,122,112,120,122,89,69,71,53,41,53,31,35,14,18)
nbpairs <- c(173,164,103,113,122,112,98,121,132,136,133,137,145,117,90,80,67,54,58,39,42,23,23)
rain <- c(67,52,88,61,32,36,72,43,92,32,86,28,57,55,66,26,28,96,48,90,86,78,87)
dat <- data.frame(nbchicks = nbchicks, 
                  nbpairs = nbpairs,
                  rain = (rain - mean(rain))/sd(rain)) # standardized rainfall
```

The data are grouped. 

## Maximum-likelihood estimation

You can get maximum likelihood estimates with:

```r
mle <- glm(cbind(nbchicks, nbpairs - nbchicks) ~ rain, 
           family = binomial("logit"), 
           data = dat)
summary(mle)
```

```
## 
## Call:
## glm(formula = cbind(nbchicks, nbpairs - nbchicks) ~ rain, family = binomial("logit"), 
##     data = dat)
## 
## Deviance Residuals: 
##     Min       1Q   Median       3Q      Max  
## -5.9630  -1.3354   0.3929   1.6168   3.8800  
## 
## Coefficients:
##             Estimate Std. Error z value Pr(>|z|)    
## (Intercept)  1.55436    0.05565  27.931   <2e-16 ***
## rain        -0.14857    0.05993  -2.479   0.0132 *  
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## (Dispersion parameter for binomial family taken to be 1)
## 
##     Null deviance: 109.25  on 22  degrees of freedom
## Residual deviance: 103.11  on 21  degrees of freedom
## AIC: 205.84
## 
## Number of Fisher Scoring iterations: 4
```

There is a negative effect of rainfall on breeding success:

```r
visreg::visreg(mle, scale = "response")
```

![](unnamed-chunk-16-1.png)<!-- -->

## Bayesian analysis with `Jags`

In Jags, we can fit the same model:

```r
# data 
dat <- list(N = 23, # nb of years
            nbchicks = nbchicks, 
            nbpairs = nbpairs,
            rain = (rain - mean(rain))/sd(rain)) # standardized rainfall

# define model
logistic <- function() {
  # likelihood
  for(i in 1:N){ # loop over years
    nbchicks[i] ~ dbin(p[i], nbpairs[i]) # binomial likelihood
    logit(p[i]) <- a + b.rain * rain[i] # prob success is linear fn of rainfall on logit scale
  }
  # priors
  a ~ dnorm(0,0.01) # intercept
  b.rain ~ dnorm(0,0.01) # slope for rainfall
}

# function that generates initial values 
inits <- function() list(a = rnorm(n = 1, mean = 0, sd = 5), 
                         b.rain = rnorm(n = 1, mean = 0, sd = 5))

# specify parameters that need to be estimated
parameters <- c("a", "b.rain")

# run Jags
bayes.jags <- jags(data = dat,
               inits = inits,
               parameters.to.save = parameters,
               model.file = logistic, 
               n.chains = 2,
               n.iter = 5000, # includes burn-in!
               n.burnin = 1000)
```

```
## Compiling model graph
##    Resolving undeclared variables
##    Allocating nodes
## Graph information:
##    Observed stochastic nodes: 23
##    Unobserved stochastic nodes: 2
##    Total graph size: 134
## 
## Initializing model
```

```r
# display results
bayes.jags
```

```
## Inference for Bugs model at "/var/folders/r7/j0wqj1k95vz8w44sdxzm986c0000gn/T//RtmpP7MZ1A/modela81d1b01bd3f.txt", fit using jags,
##  2 chains, each with 5000 iterations (first 1000 discarded), n.thin = 4
##  n.sims = 2000 iterations saved
##          mu.vect sd.vect    2.5%     25%     50%     75%   97.5%  Rhat n.eff
## a          1.546   0.163   1.439   1.514   1.552   1.588   1.657 1.182  2000
## b.rain    -0.152   0.121  -0.275  -0.190  -0.148  -0.106  -0.033 1.129  2000
## deviance 213.612 330.910 201.899 202.434 203.197 204.689 209.169 1.121  2000
## 
## For each parameter, n.eff is a crude measure of effective sample size,
## and Rhat is the potential scale reduction factor (at convergence, Rhat=1).
## 
## DIC info (using the rule, pD = var(deviance)/2)
## pD = 54753.9 and DIC = 54967.5
## DIC is an estimate of expected predictive error (lower deviance is better).
```

## Bayesian analysis with `brms`

With `brms`, we write:

```r
bayes.brms <- brm(nbchicks | trials(nbpairs) ~ rain, 
                  family = binomial("logit"), 
                  data = dat,
                  chains = 2, # nb of chains
                  iter = 5000, # nb of iterations, including burnin
                  warmup = 1000, # burnin
                  thin = 1)
```

```
## Running /Library/Frameworks/R.framework/Resources/bin/R CMD SHLIB foo.c
## clang -mmacosx-version-min=10.13 -I"/Library/Frameworks/R.framework/Resources/include" -DNDEBUG   -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/Rcpp/include/"  -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/"  -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/unsupported"  -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/BH/include" -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/StanHeaders/include/src/"  -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/StanHeaders/include/"  -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppParallel/include/"  -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/rstan/include" -DEIGEN_NO_DEBUG  -DBOOST_DISABLE_ASSERTS  -DBOOST_PENDING_INTEGER_LOG2_HPP  -DSTAN_THREADS  -DBOOST_NO_AUTO_PTR  -include '/Library/Frameworks/R.framework/Versions/4.1/Resources/library/StanHeaders/include/stan/math/prim/mat/fun/Eigen.hpp'  -D_REENTRANT -DRCPP_PARALLEL_USE_TBB=1   -I/usr/local/include   -fPIC  -Wall -g -O2  -c foo.c -o foo.o
## In file included from <built-in>:1:
## In file included from /Library/Frameworks/R.framework/Versions/4.1/Resources/library/StanHeaders/include/stan/math/prim/mat/fun/Eigen.hpp:13:
## In file included from /Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/Eigen/Dense:1:
## In file included from /Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/Eigen/Core:88:
## /Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/Eigen/src/Core/util/Macros.h:628:1: error: unknown type name 'namespace'
## namespace Eigen {
## ^
## /Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/Eigen/src/Core/util/Macros.h:628:16: error: expected ';' after top level declarator
## namespace Eigen {
##                ^
##                ;
## In file included from <built-in>:1:
## In file included from /Library/Frameworks/R.framework/Versions/4.1/Resources/library/StanHeaders/include/stan/math/prim/mat/fun/Eigen.hpp:13:
## In file included from /Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/Eigen/Dense:1:
## /Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/Eigen/Core:96:10: fatal error: 'complex' file not found
## #include <complex>
##          ^~~~~~~~~
## 3 errors generated.
## make: *** [foo.o] Error 1
## 
## SAMPLING FOR MODEL '70a68f0c53365768853c2791038e6352' NOW (CHAIN 1).
## Chain 1: 
## Chain 1: Gradient evaluation took 4.7e-05 seconds
## Chain 1: 1000 transitions using 10 leapfrog steps per transition would take 0.47 seconds.
## Chain 1: Adjust your expectations accordingly!
## Chain 1: 
## Chain 1: 
## Chain 1: Iteration:    1 / 5000 [  0%]  (Warmup)
## Chain 1: Iteration:  500 / 5000 [ 10%]  (Warmup)
## Chain 1: Iteration: 1000 / 5000 [ 20%]  (Warmup)
## Chain 1: Iteration: 1001 / 5000 [ 20%]  (Sampling)
## Chain 1: Iteration: 1500 / 5000 [ 30%]  (Sampling)
## Chain 1: Iteration: 2000 / 5000 [ 40%]  (Sampling)
## Chain 1: Iteration: 2500 / 5000 [ 50%]  (Sampling)
## Chain 1: Iteration: 3000 / 5000 [ 60%]  (Sampling)
## Chain 1: Iteration: 3500 / 5000 [ 70%]  (Sampling)
## Chain 1: Iteration: 4000 / 5000 [ 80%]  (Sampling)
## Chain 1: Iteration: 4500 / 5000 [ 90%]  (Sampling)
## Chain 1: Iteration: 5000 / 5000 [100%]  (Sampling)
## Chain 1: 
## Chain 1:  Elapsed Time: 0.035213 seconds (Warm-up)
## Chain 1:                0.131663 seconds (Sampling)
## Chain 1:                0.166876 seconds (Total)
## Chain 1: 
## 
## SAMPLING FOR MODEL '70a68f0c53365768853c2791038e6352' NOW (CHAIN 2).
## Chain 2: 
## Chain 2: Gradient evaluation took 1.4e-05 seconds
## Chain 2: 1000 transitions using 10 leapfrog steps per transition would take 0.14 seconds.
## Chain 2: Adjust your expectations accordingly!
## Chain 2: 
## Chain 2: 
## Chain 2: Iteration:    1 / 5000 [  0%]  (Warmup)
## Chain 2: Iteration:  500 / 5000 [ 10%]  (Warmup)
## Chain 2: Iteration: 1000 / 5000 [ 20%]  (Warmup)
## Chain 2: Iteration: 1001 / 5000 [ 20%]  (Sampling)
## Chain 2: Iteration: 1500 / 5000 [ 30%]  (Sampling)
## Chain 2: Iteration: 2000 / 5000 [ 40%]  (Sampling)
## Chain 2: Iteration: 2500 / 5000 [ 50%]  (Sampling)
## Chain 2: Iteration: 3000 / 5000 [ 60%]  (Sampling)
## Chain 2: Iteration: 3500 / 5000 [ 70%]  (Sampling)
## Chain 2: Iteration: 4000 / 5000 [ 80%]  (Sampling)
## Chain 2: Iteration: 4500 / 5000 [ 90%]  (Sampling)
## Chain 2: Iteration: 5000 / 5000 [100%]  (Sampling)
## Chain 2: 
## Chain 2:  Elapsed Time: 0.033715 seconds (Warm-up)
## Chain 2:                0.15264 seconds (Sampling)
## Chain 2:                0.186355 seconds (Total)
## Chain 2:
```

Display results:

```r
bayes.brms
```

```
##  Family: binomial 
##   Links: mu = logit 
## Formula: nbchicks | trials(nbpairs) ~ rain 
##    Data: dat (Number of observations: 23) 
##   Draws: 2 chains, each with iter = 5000; warmup = 1000; thin = 1;
##          total post-warmup draws = 8000
## 
## Population-Level Effects: 
##           Estimate Est.Error l-95% CI u-95% CI Rhat Bulk_ESS Tail_ESS
## Intercept     1.56      0.06     1.45     1.67 1.00     6907     5043
## rain         -0.15      0.06    -0.27    -0.03 1.00     7193     5067
## 
## Draws were sampled using sampling(NUTS). For each parameter, Bulk_ESS
## and Tail_ESS are effective sample size measures, and Rhat is the potential
## scale reduction factor on split chains (at convergence, Rhat = 1).
```

Visualize:

```r
plot(bayes.brms)
```

![](unnamed-chunk-20-1.png)<!-- -->

We can also calculate the posterior probability of the rainfall effect being below zero with the `hypothesis()` function:

```r
hypothesis(bayes.brms, 'rain < 0')
```

```
## Hypothesis Tests for class b:
##   Hypothesis Estimate Est.Error CI.Lower CI.Upper Evid.Ratio Post.Prob Star
## 1 (rain) < 0    -0.15      0.06    -0.25    -0.05     116.65      0.99    *
## ---
## 'CI': 90%-CI for one-sided and 95%-CI for two-sided hypotheses.
## '*': For one-sided hypotheses, the posterior probability exceeds 95%;
## for two-sided hypotheses, the value tested against lies outside the 95%-CI.
## Posterior probabilities of point hypotheses assume equal prior probabilities.
```

These results confirm the important negative effect of rainfall on breeding success.

# Linear mixed model

In this example, we have several measurements for 33 Mediterranean plant species, specifically number of seeds and biomass. We ask whether there is a linear relationship between these two variables that would hold for all species. 

Read in data, directly from the course website:

```r
df <- read_csv2("https://raw.githubusercontent.com/oliviergimenez/bayesian-stats-with-R/master/slides/dat/VMG.csv")
df
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":["Sp"],"name":[1],"type":["chr"],"align":["left"]},{"label":["NGrTotest"],"name":[2],"type":["dbl"],"align":["right"]},{"label":["Vm"],"name":[3],"type":["chr"],"align":["left"]}],"data":[{"1":"AGREUP","2":"5.220000e+02","3":"2.7402"},{"1":"AGREUP","2":"9.280000e+02","3":"5.7107"},{"1":"AGREUP","2":"2.349000e+03","3":"3.7874"},{"1":"AGREUP","2":"1.261500e+04","3":"4.208"},{"1":"AGREUP","2":"9.135000e+03","3":"4.6764"},{"1":"AGREUP","2":"1.189000e+03","3":"4.5658"},{"1":"AGREUP","2":"9.280000e+02","3":"5.4989"},{"1":"AGREUP","2":"1.232500e+04","3":"6.0688"},{"1":"AGREUP","2":"2.044500e+04","3":"5.2587"},{"1":"AGREUP","2":"1.145500e+04","3":"5.4009"},{"1":"AGREUP","2":"7.395000e+03","3":"2.4289"},{"1":"AGREUP","2":"1.812500e+04","3":"5.7908"},{"1":"AGREUP","2":"1.116500e+04","3":"4.3584"},{"1":"AGREUP","2":"8.410000e+02","3":"3.0194"},{"1":"ARESER","2":"9.888138e+06","3":"0.2589"},{"1":"ARESER","2":"7.378453e+14","3":"0.2509"},{"1":"ARESER","2":"5.370114e+14","3":"0.1953"},{"1":"ARESER","2":"8.516877e+14","3":"0.072"},{"1":"ARESER","2":"1.983254e+14","3":"0.0922"},{"1":"ARESER","2":"1.026845e+07","3":"0.0848"},{"1":"ARESER","2":"3.905707e+14","3":"0.0502"},{"1":"ARESER","2":"2.842153e+14","3":"0.03"},{"1":"ARESER","2":"7.697224e+14","3":"0.0929"},{"1":"ARESER","2":"1.269531e+14","3":"0.0288"},{"1":"ARESER","2":"4.860176e+14","3":"0.0539"},{"1":"ARESER","2":"3.122590e+13","3":"0.0432"},{"1":"ARIROT","2":"7.740000e+02","3":"2.5837"},{"1":"ARIROT","2":"7.740000e+02","3":"2.9822"},{"1":"ARIROT","2":"3.870000e+02","3":"4.8398"},{"1":"ARIROT","2":"7.740000e+02","3":"3.8271"},{"1":"ARIROT","2":"1.548000e+03","3":"3.2125"},{"1":"ARIROT","2":"1.548000e+03","3":"4.0537"},{"1":"ARIROT","2":"7.740000e+02","3":"3.7995"},{"1":"AVEBAR","2":"1.885159e+06","3":"0.7485"},{"1":"AVEBAR","2":"4.497265e+06","3":"0.3794"},{"1":"AVEBAR","2":"2.679216e+07","3":"0.9408"},{"1":"AVEBAR","2":"1.534996e+06","3":"1.0311"},{"1":"AVEBAR","2":"1.396726e+09","3":"6.2405"},{"1":"AVEBAR","2":"1.540947e+07","3":"6.3843"},{"1":"AVEBAR","2":"1.851852e+08","3":"0.8972"},{"1":"AVEBAR","2":"1.545312e+08","3":"1.3303"},{"1":"AVEBAR","2":"1.908618e+08","3":"12.0858"},{"1":"AVEBAR","2":"5.312807e+08","3":"1.8016"},{"1":"AVEBAR","2":"3.087444e+08","3":"3.8879"},{"1":"AVEBAR","2":"7.085170e+05","3":"0.4081"},{"1":"AVEBAR","2":"9.622968e+07","3":"0.4865"},{"1":"AVEBAR","2":"1.956194e+08","3":"1.0913"},{"1":"AVEBAR","2":"6.710838e+07","3":"0.3691"},{"1":"BRAPHO","2":"2.500000e+01","3":"1.4411"},{"1":"BRAPHO","2":"8.000000e+00","3":"3.3895"},{"1":"BRAPHO","2":"1.200000e+01","3":"8.7321"},{"1":"BRAPHO","2":"3.200000e+01","3":"0.7551"},{"1":"BRAPHO","2":"2.700000e+01","3":"2.84"},{"1":"BRAPHO","2":"1.800000e+01","3":"1.7812"},{"1":"BRAPHO","2":"4.000000e+00","3":"2.2782"},{"1":"BRAPHO","2":"7.000000e+00","3":"3.0333"},{"1":"BRAPHO","2":"2.700000e+01","3":"4.5069"},{"1":"BRAPHO","2":"3.000000e+00","3":"5.2177"},{"1":"BRAPHO","2":"1.000000e+00","3":"2.6946"},{"1":"BRAPHO","2":"1.500000e+01","3":"2.9896"},{"1":"BRAPHO","2":"1.500000e+01","3":"2.6708"},{"1":"BRAPHO","2":"5.000000e+00","3":"7.183"},{"1":"BRAPHO","2":"4.000000e+00","3":"2.3428"},{"1":"BRAPHO","2":"7.000000e+00","3":"1.0896"},{"1":"BRAPHO","2":"5.000000e+00","3":"2.6775"},{"1":"BRAPHO","2":"1.000000e+01","3":"1.7357"},{"1":"BRAPHO","2":"5.000000e+00","3":"1.3361"},{"1":"BRAPHO","2":"1.400000e+01","3":"2.0915"},{"1":"BROERE","2":"3.000000e+00","3":"1.4552"},{"1":"BROERE","2":"2.300000e+01","3":"7.0985"},{"1":"BROERE","2":"1.000000e+00","3":"1.8038"},{"1":"BROERE","2":"1.100000e+01","3":"1.165"},{"1":"BROERE","2":"6.800000e+01","3":"3.2737"},{"1":"BROERE","2":"1.800000e+01","3":"1.5305"},{"1":"BROERE","2":"9.000000e+00","3":"2.674"},{"1":"BROERE","2":"1.300000e+01","3":"1.4999"},{"1":"BROERE","2":"1.000000e+00","3":"1.3962"},{"1":"BROERE","2":"1.100000e+01","3":"2.6229"},{"1":"BROERE","2":"7.000000e+00","3":"2.2209"},{"1":"BROERE","2":"1.400000e+01","3":"1.8808"},{"1":"BROERE","2":"1.700000e+01","3":"1.972"},{"1":"BROERE","2":"3.000000e+00","3":"4.6314"},{"1":"BROERE","2":"1.300000e+01","3":"3.9903"},{"1":"BROMAD","2":"8.000000e+00","3":"0.0287"},{"1":"BROMAD","2":"1.000000e+01","3":"0.0408"},{"1":"BROMAD","2":"2.100000e+01","3":"0.0573"},{"1":"BROMAD","2":"2.000000e+01","3":"0.0653"},{"1":"BROMAD","2":"1.600000e+01","3":"0.0557"},{"1":"BROMAD","2":"1.600000e+01","3":"0.0448"},{"1":"BROMAD","2":"2.000000e+01","3":"0.0577"},{"1":"BROMAD","2":"2.500000e+01","3":"0.0811"},{"1":"BROMAD","2":"3.700000e+01","3":"0.2731"},{"1":"BROMAD","2":"1.800000e+01","3":"0.0804"},{"1":"BROMAD","2":"3.800000e+01","3":"0.1597"},{"1":"BROMAD","2":"8.000000e+00","3":"0.0508"},{"1":"BROMAD","2":"5.000000e+00","3":"0.0227"},{"1":"BROMAD","2":"1.700000e+01","3":"0.041"},{"1":"BROMAD","2":"5.000000e+00","3":"0.0585"},{"1":"BROMAD","2":"7.185522e+06","3":"0.3702"},{"1":"BROMAD","2":"5.950135e+07","3":"0.3718"},{"1":"BROMAD","2":"1.400000e+01","3":"0.094"},{"1":"BROMAD","2":"5.000000e+01","3":"0.2454"},{"1":"BROMAD","2":"7.000000e+00","3":"0.0322"},{"1":"BROMAD","2":"1.100000e+01","3":"0.035"},{"1":"BROMAD","2":"2.700000e+01","3":"0.1002"},{"1":"BROMAD","2":"1.000000e+01","3":"0.1121"},{"1":"BROMAD","2":"4.800000e+01","3":"0.1931"},{"1":"BROMAD","2":"3.100000e+01","3":"0.0797"},{"1":"BROMAD","2":"5.900000e+01","3":"0.1937"},{"1":"BROMAD","2":"2.700000e+01","3":"0.0921"},{"1":"CALNEP","2":"1.943000e+03","3":"0.6494"},{"1":"CALNEP","2":"1.453900e+04","3":"3.5915"},{"1":"CALNEP","2":"3.316500e+04","3":"3.8162"},{"1":"CALNEP","2":"2.010000e+02","3":"1.3129"},{"1":"CALNEP","2":"2.043500e+04","3":"0.8645"},{"1":"CALNEP","2":"2.546000e+03","3":"5.0374"},{"1":"CALNEP","2":"6.867500e+04","3":"1.3146"},{"1":"CALNEP","2":"2.110500e+04","3":"1.4455"},{"1":"CALNEP","2":"2.613000e+03","3":"1.5333"},{"1":"CALNEP","2":"2.211000e+03","3":"1.879"},{"1":"CALNEP","2":"1.641500e+04","3":"1.2037"},{"1":"CALNEP","2":"1.340000e+02","3":"3.0969"},{"1":"CENASP","2":"1.055600e+04","3":"1.9068"},{"1":"CENASP","2":"1.206400e+04","3":"4.2394"},{"1":"CENASP","2":"1.809600e+04","3":"3.7426"},{"1":"CENASP","2":"1.658800e+04","3":"2.0106"},{"1":"CENASP","2":"5.956600e+04","3":"11.937"},{"1":"CENASP","2":"2.789800e+04","3":"6.6217"},{"1":"CENASP","2":"1.809600e+04","3":"3.1996"},{"1":"CENASP","2":"1.131000e+03","3":"2.3526"},{"1":"CENASP","2":"1.055600e+04","3":"1.3443"},{"1":"CENASP","2":"1.206400e+04","3":"3.2106"},{"1":"CONARV","2":"6.560000e+02","3":"0.8748"},{"1":"CONARV","2":"2.296000e+03","3":"1.0244"},{"1":"CONARV","2":"6.560000e+02","3":"0.4904"},{"1":"CONARV","2":"9.840000e+02","3":"0.4121"},{"1":"CONARV","2":"3.280000e+02","3":"0.4221"},{"1":"CONARV","2":"1.968000e+03","3":"1.5497"},{"1":"CONARV","2":"2.624000e+03","3":"0.4686"},{"1":"CONARV","2":"3.936000e+03","3":"0.5614"},{"1":"CONARV","2":"4.920000e+02","3":"0.7569"},{"1":"CONARV","2":"5.904000e+03","3":"1.4327"},{"1":"CONARV","2":"1.312000e+03","3":"0.1692"},{"1":"CONCAN","2":"1.098020e+05","3":"1.823"},{"1":"CONCAN","2":"1.771000e+04","3":"3.965"},{"1":"CONCAN","2":"1.931160e+05","3":"2.929"},{"1":"CONCAN","2":"1.016400e+04","3":"1.879"},{"1":"CONCAN","2":"2.882880e+05","3":"4.749"},{"1":"CONCAN","2":"2.858240e+05","3":"5.477"},{"1":"CONCAN","2":"1.182720e+05","3":"1.893"},{"1":"CONCAN","2":"6.375600e+04","3":"0.605"},{"1":"CONCAN","2":"7.068600e+04","3":"0.003"},{"1":"CONCAN","2":"8.870400e+04","3":"1.795"},{"1":"CONCAN","2":"1.787940e+05","3":"3.167"},{"1":"CONCAN","2":"1.067220e+05","3":"1.739"},{"1":"CONCAN","2":"1.321320e+05","3":"2.691"},{"1":"CONCAN","2":"1.466080e+05","3":"2.201"},{"1":"CONCAN","2":"1.016400e+04","3":"0.941"},{"1":"CONCAN","2":"1.111110e+05","3":"2.1331"},{"1":"CONCAN","2":"3.172785e+06","3":"3.5198"},{"1":"CONCAN","2":"1.552320e+05","3":"2.7188"},{"1":"CONCAN","2":"3.094245e+06","3":"4.9485"},{"1":"CONCAN","2":"9.378600e+04","3":"0.9582"},{"1":"CONSUM","2":"2.793000e+04","3":"7.884"},{"1":"CONSUM","2":"1.764000e+04","3":"5.827"},{"1":"CONSUM","2":"1.849260e+05","3":"5.64"},{"1":"CONSUM","2":"6.585600e+04","3":"14.242"},{"1":"CONSUM","2":"3.372180e+05","3":"11.624"},{"1":"CONSUM","2":"6.043050e+11","3":"11.0929"},{"1":"CONSUM","2":"1.360787e+11","3":"4.6419"},{"1":"CONSUM","2":"4.655741e+09","3":"9.3029"},{"1":"CONSUM","2":"2.747341e+11","3":"3.7676"},{"1":"CONSUM","2":"3.394929e+11","3":"11.7619"},{"1":"CREFOE","2":"5.450000e+02","3":"0.8151"},{"1":"CREFOE","2":"6.195000e+03","3":"1.2244"},{"1":"CREFOE","2":"7.615000e+03","3":"0.9312"},{"1":"CREFOE","2":"5.022500e+04","3":"0.5408"},{"1":"CREFOE","2":"1.051750e+05","3":"1.5945"},{"1":"CREFOE","2":"6.385000e+03","3":"0.8848"},{"1":"CREFOE","2":"6.295000e+03","3":"1.0231"},{"1":"CREFOE","2":"6.675000e+03","3":"0.6402"},{"1":"CREFOE","2":"2.257500e+04","3":"0.4692"},{"1":"CREFOE","2":"1.497000e+03","3":"2.4955"},{"1":"CREFOE","2":"5.145000e+03","3":"0.8226"},{"1":"CREFOE","2":"4.280000e+02","3":"0.43"},{"1":"CREFOE","2":"6.872500e+04","3":"0.4259"},{"1":"CYNDAC","2":"6.944858e+14","3":"0.5576"},{"1":"CYNDAC","2":"2.137188e+14","3":"0.9056"},{"1":"CYNDAC","2":"1.628894e+14","3":"1.1958"},{"1":"CYNDAC","2":"9.478765e+14","3":"0.3839"},{"1":"CYNDAC","2":"3.042291e+14","3":"0.5038"},{"1":"CYNDAC","2":"1.214020e+14","3":"0.4434"},{"1":"CYNDAC","2":"3.871472e+14","3":"0.7369"},{"1":"CYNDAC","2":"1.455768e+14","3":"0.6764"},{"1":"CYNDAC","2":"4.066404e+14","3":"0.8645"},{"1":"CYNDAC","2":"1.658677e+14","3":"0.6768"},{"1":"CYNDAC","2":"9.252856e+14","3":"0.6439"},{"1":"CYNDAC","2":"7.542087e+13","3":"0.465"},{"1":"DACGLO","2":"1.245469e+14","3":"2.822"},{"1":"DACGLO","2":"1.917928e+14","3":"5.925"},{"1":"DACGLO","2":"8.404472e+14","3":"4.442"},{"1":"DACGLO","2":"1.962699e+14","3":"1.322"},{"1":"DACGLO","2":"6.064351e+14","3":"1.607"},{"1":"DACGLO","2":"4.313892e+14","3":"5.113"},{"1":"DACGLO","2":"4.129153e+14","3":"3.258"},{"1":"DACGLO","2":"4.774426e+14","3":"2.265"},{"1":"DACGLO","2":"1.312217e+13","3":"4.858"},{"1":"DACGLO","2":"9.394479e+14","3":"3.5832"},{"1":"DACGLO","2":"7.719279e+14","3":"5.422"},{"1":"DACGLO","2":"1.296418e+13","3":"3.696"},{"1":"DACGLO","2":"3.604338e+14","3":"3.893"},{"1":"DAUCAR","2":"6.974874e+14","3":"2.7065"},{"1":"DAUCAR","2":"2.789950e+14","3":"7.2954"},{"1":"DAUCAR","2":"6.277387e+14","3":"6.7922"},{"1":"DAUCAR","2":"6.974874e+14","3":"5.8217"},{"1":"DAUCAR","2":"5.579899e+14","3":"5.2203"},{"1":"DAUCAR","2":"4.882412e+14","3":"4.2592"},{"1":"DAUCAR","2":"7.672361e+14","3":"3.485"},{"1":"DAUCAR","2":"3.487437e+14","3":"3.4059"},{"1":"DAUCAR","2":"4.882412e+14","3":"5.6538"},{"1":"DAUCAR","2":"2.789950e+14","3":"1.6279"},{"1":"DAUCAR","2":"4.184924e+13","3":"6.3016"},{"1":"DAUCAR","2":"2.789950e+14","3":"2.2578"},{"1":"DAUCAR","2":"1.325226e+14","3":"13.8315"},{"1":"DAUCAR","2":"2.789950e+14","3":"2.5385"},{"1":"DAUCAR","2":"4.184924e+13","3":"2.4443"},{"1":"DAUCAR","2":"2.092462e+14","3":"0.6998"},{"1":"DIPFUL","2":"5.991644e+13","3":"9.26"},{"1":"DIPFUL","2":"9.300481e+14","3":"7.963"},{"1":"DIPFUL","2":"4.492366e+13","3":"7.141"},{"1":"DIPFUL","2":"2.405171e+14","3":"23.364"},{"1":"DIPFUL","2":"2.030262e+14","3":"24.003"},{"1":"DIPFUL","2":"1.657848e+14","3":"14.728"},{"1":"DIPFUL","2":"7.631866e+14","3":"8.506"},{"1":"DIPFUL","2":"7.541823e+14","3":"7.253"},{"1":"DIPFUL","2":"8.924796e+14","3":"12.1349"},{"1":"DIPFUL","2":"4.966303e+14","3":"38.73"},{"1":"DIPFUL","2":"3.788564e+14","3":"29.417"},{"1":"DIPFUL","2":"5.835986e+14","3":"31.51"},{"1":"DIPFUL","2":"3.260063e+14","3":"4.94"},{"1":"DIPFUL","2":"3.170862e+14","3":"24.603"},{"1":"DIPFUL","2":"1.079704e+14","3":"9.922"},{"1":"DIPFUL","2":"1.039471e+14","3":"9.677"},{"1":"DIPFUL","2":"3.254672e+14","3":"30.433"},{"1":"DIPFUL","2":"7.365023e+14","3":"12.038"},{"1":"DIPFUL","2":"3.825576e+14","3":"29.728"},{"1":"DIPFUL","2":"2.083637e+14","3":"23.141"},{"1":"DIPFUL","2":"2.314471e+14","3":"25.903"},{"1":"EROCIC","2":"3.600000e+02","3":"2.2693"},{"1":"EROCIC","2":"1.035000e+03","3":"13.3441"},{"1":"EROCIC","2":"2.400000e+02","3":"2.4296"},{"1":"EROCIC","2":"7.300000e+02","3":"8.7793"},{"1":"EROCIC","2":"3.150000e+02","3":"3.4946"},{"1":"EROCIC","2":"2.000000e+02","3":"1.8299"},{"1":"EROCIC","2":"3.350000e+02","3":"4.4712"},{"1":"EROCIC","2":"8.200000e+02","3":"8.0342"},{"1":"EROCIC","2":"2.800000e+02","3":"2.8258"},{"1":"EROCIC","2":"2.650000e+02","3":"1.7212"},{"1":"EROCIC","2":"7.050000e+02","3":"5.7427"},{"1":"EROCIC","2":"2.050000e+02","3":"1.3254"},{"1":"EROCIC","2":"1.750000e+02","3":"0.8842"},{"1":"EROCIC","2":"1.350000e+02","3":"0.8806"},{"1":"GERROT","2":"3.090000e+02","3":"0.6574"},{"1":"GERROT","2":"1.140000e+02","3":"0.1645"},{"1":"GERROT","2":"9.700000e+01","3":"0.1489"},{"1":"GERROT","2":"1.910000e+02","3":"0.3903"},{"1":"GERROT","2":"2.790000e+02","3":"0.4476"},{"1":"GERROT","2":"3.160000e+02","3":"0.5542"},{"1":"GERROT","2":"2.120000e+02","3":"0.311"},{"1":"GERROT","2":"1.180000e+02","3":"0.1774"},{"1":"GERROT","2":"2.530000e+02","3":"0.3745"},{"1":"GERROT","2":"1.540000e+02","3":"0.1993"},{"1":"GERROT","2":"1.360000e+02","3":"0.268"},{"1":"GERROT","2":"2.490000e+02","3":"0.5365"},{"1":"GERROT","2":"3.800000e+02","3":"0.6972"},{"1":"GERROT","2":"4.750000e+02","3":"0.838"},{"1":"GERROT","2":"1.760000e+02","3":"0.3191"},{"1":"INUCON","2":"2.547462e+10","3":"3.4002"},{"1":"INUCON","2":"8.855462e+10","3":"6.9655"},{"1":"INUCON","2":"3.093346e+11","3":"3.1757"},{"1":"INUCON","2":"2.972038e+11","3":"1.5498"},{"1":"INUCON","2":"2.122885e+11","3":"13.2155"},{"1":"INUCON","2":"4.912962e+11","3":"5.215"},{"1":"INUCON","2":"6.429308e+10","3":"6.7267"},{"1":"LOLITA","2":"1.370700e+04","3":"0.1732"},{"1":"LOLITA","2":"1.370700e+04","3":"0.2786"},{"1":"LOLITA","2":"7.234250e+05","3":"1.197"},{"1":"LOLITA","2":"2.132200e+04","3":"0.2314"},{"1":"LOLITA","2":"1.751450e+05","3":"0.2886"},{"1":"LOLITA","2":"7.081950e+05","3":"0.8304"},{"1":"LOLITA","2":"6.548900e+04","3":"0.7924"},{"1":"LOLITA","2":"3.274450e+05","3":"0.4729"},{"1":"LOLITA","2":"3.046000e+03","3":"0.6055"},{"1":"LOLITA","2":"2.056050e+05","3":"0.2231"},{"1":"LOLITA","2":"6.092000e+03","3":"0.8283"},{"1":"LOLITA","2":"4.949750e+05","3":"0.5897"},{"1":"LOLITA","2":"7.691150e+05","3":"1.3697"},{"1":"LOLITA","2":"3.046000e+03","3":"0.2796"},{"1":"LOLITA","2":"6.320450e+05","3":"1.3046"},{"1":"MEDLUP","2":"5.534226e+14","3":"0.2427"},{"1":"MEDLUP","2":"1.555585e+14","3":"0.0834"},{"1":"MEDLUP","2":"2.729915e+14","3":"0.1296"},{"1":"MEDLUP","2":"8.481451e+14","3":"0.4678"},{"1":"MEDLUP","2":"1.676752e+14","3":"0.8808"},{"1":"MEDLUP","2":"4.266047e+14","3":"0.1615"},{"1":"MEDLUP","2":"1.016326e+14","3":"0.4961"},{"1":"MEDLUP","2":"1.843326e+14","3":"0.1084"},{"1":"MEDLUP","2":"7.252031e+14","3":"0.0753"},{"1":"MEDLUP","2":"1.272668e+13","3":"0.0558"},{"1":"MEDLUP","2":"1.843326e+14","3":"0.0815"},{"1":"MEDLUP","2":"2.135262e+14","3":"0.0751"},{"1":"MEDMIN","2":"4.004000e+03","3":"0.6305"},{"1":"MEDMIN","2":"5.324000e+03","3":"0.7074"},{"1":"MEDMIN","2":"1.188000e+03","3":"0.1144"},{"1":"MEDMIN","2":"3.520000e+02","3":"0.029"},{"1":"MEDMIN","2":"2.552000e+03","3":"0.2778"},{"1":"MEDMIN","2":"2.464000e+03","3":"0.2053"},{"1":"MEDMIN","2":"6.160000e+02","3":"0.063"},{"1":"MEDMIN","2":"1.232000e+03","3":"0.184"},{"1":"MEDMIN","2":"8.360000e+02","3":"0.1834"},{"1":"MEDMIN","2":"2.200000e+01","3":"0.0484"},{"1":"MEDMIN","2":"1.408000e+03","3":"0.3134"},{"1":"ORLGRA","2":"2.097000e+03","3":"0.3644"},{"1":"ORLGRA","2":"2.923000e+03","3":"0.4078"},{"1":"ORLGRA","2":"4.555000e+03","3":"0.945"},{"1":"ORLGRA","2":"6.168000e+03","3":"1.2278"},{"1":"ORLGRA","2":"2.800000e+01","3":"0.6491"},{"1":"ORLGRA","2":"1.001000e+03","3":"1.3631"},{"1":"ORLGRA","2":"5.400000e+01","3":"1.285"},{"1":"ORLGRA","2":"3.610000e+02","3":"0.6316"},{"1":"ORLGRA","2":"1.700000e+01","3":"0.2334"},{"1":"ORLGRA","2":"3.323000e+03","3":"0.3953"},{"1":"ORLGRA","2":"1.900000e+01","3":"0.3508"},{"1":"ORLGRA","2":"2.000000e+01","3":"0.4501"},{"1":"ORLGRA","2":"8.000000e+00","3":"0.2085"},{"1":"ORLGRA","2":"2.600000e+01","3":"0.6552"},{"1":"ORLGRA","2":"2.155000e+03","3":"0.2435"},{"1":"ORLGRA","2":"5.633000e+03","3":"1.0596"},{"1":"ORLGRA","2":"1.881000e+03","3":"0.3286"},{"1":"ORLGRA","2":"3.052000e+03","3":"0.5003"},{"1":"ORLGRA","2":"1.323100e+04","3":"2.2946"},{"1":"ORLGRA","2":"2.126000e+03","3":"0.3741"},{"1":"ORLGRA","2":"3.181000e+03","3":"0.5459"},{"1":"ORLGRA","2":"2.600000e+01","3":"0.5727"},{"1":"ORLGRA","2":"2.900000e+01","3":"0.7562"},{"1":"ORLGRA","2":"1.284000e+03","3":"0.2016"},{"1":"ORLGRA","2":"1.104900e+04","3":"2.2006"},{"1":"ORLGRA","2":"7.607000e+03","3":"0.8084"},{"1":"ORLGRA","2":"5.481000e+03","3":"1.0416"},{"1":"PICHIE","2":"3.590440e+05","3":"3.6551"},{"1":"PICHIE","2":"8.719640e+05","3":"4.5974"},{"1":"PICHIE","2":"7.693800e+04","3":"2.8561"},{"1":"PICHIE","2":"1.025840e+05","3":"5.3615"},{"1":"PICHIE","2":"8.206720e+05","3":"4.5136"},{"1":"PICHIE","2":"1.538760e+05","3":"3.5334"},{"1":"PICHIE","2":"1.077132e+06","3":"5.2522"},{"1":"PICHIE","2":"3.590440e+05","3":"3.051"},{"1":"PICHIE","2":"1.333592e+06","3":"6.1755"},{"1":"PICHIE","2":"1.179716e+06","3":"9.8372"},{"1":"PICHIE","2":"8.206720e+05","3":"2.902"},{"1":"PICHIE","2":"3.077520e+05","3":"4.0246"},{"1":"PICHIE","2":"1.025840e+05","3":"7.6592"},{"1":"PICHIE","2":"1.538760e+05","3":"1.9749"},{"1":"PICHIE","2":"1.846512e+06","3":"7.5188"},{"1":"PICHIE","2":"1.025840e+05","3":"4.7883"},{"1":"PICHIE","2":"9.232560e+05","3":"3.688"},{"1":"PICHIE","2":"4.616280e+05","3":"2.4499"},{"1":"PICHIE","2":"4.616280e+05","3":"4.4144"},{"1":"PICHIE","2":"2.102972e+06","3":"9.5132"},{"1":"PICHIE","2":"4.616280e+05","3":"3.6414"},{"1":"PICHIE","2":"2.051680e+05","3":"3.081"},{"1":"PICHIE","2":"1.692636e+06","3":"10.6821"},{"1":"SANMIN","2":"1.000000e+01","3":"0.2655"},{"1":"SANMIN","2":"2.000000e+01","3":"0.8077"},{"1":"SANMIN","2":"6.044100e+04","3":"0.95"},{"1":"SANMIN","2":"3.244100e+04","3":"0.2544"},{"1":"SANMIN","2":"2.100000e+01","3":"0.8371"},{"1":"SANMIN","2":"2.744100e+04","3":"0.5838"},{"1":"SANMIN","2":"6.600000e+01","3":"1.2204"},{"1":"SANMIN","2":"2.011124e+14","3":"3.7847"},{"1":"SANMIN","2":"3.959632e+14","3":"1.3835"},{"1":"SANMIN","2":"5.244100e+04","3":"0.5912"},{"1":"SANMIN","2":"3.444100e+04","3":"0.3044"},{"1":"SANMIN","2":"3.744100e+04","3":"0.8692"},{"1":"SANMIN","2":"1.100000e+01","3":"0.6444"},{"1":"SEDNIC","2":"3.764507e+14","3":"2.3502"},{"1":"SEDNIC","2":"4.897700e+04","3":"0.3377"},{"1":"SEDNIC","2":"8.354900e+04","3":"0.5203"},{"1":"SEDNIC","2":"2.160750e+05","3":"0.139"},{"1":"SEDNIC","2":"5.473900e+04","3":"0.8127"},{"1":"SEDNIC","2":"8.354900e+04","3":"0.5362"},{"1":"SEDNIC","2":"5.617950e+05","3":"0.4823"},{"1":"SEDNIC","2":"3.457200e+04","3":"0.2996"},{"1":"SEDNIC","2":"4.177450e+05","3":"0.195"},{"1":"SEDNIC","2":"5.906050e+05","3":"0.4522"},{"1":"TORJAP","2":"9.459805e+14","3":"0.8427"},{"1":"TORJAP","2":"5.424562e+14","3":"0.4478"},{"1":"TORJAP","2":"1.338922e+14","3":"0.5202"},{"1":"TORJAP","2":"1.447315e+14","3":"1.0882"},{"1":"TORJAP","2":"3.501219e+14","3":"1.6174"},{"1":"TORJAP","2":"1.733331e+14","3":"1.678"},{"1":"TORJAP","2":"2.042334e+14","3":"1.2695"},{"1":"TORJAP","2":"1.447315e+14","3":"1.2443"},{"1":"TORJAP","2":"9.459805e+14","3":"1.3019"},{"1":"TORJAP","2":"1.338922e+14","3":"0.1205"},{"1":"TORJAP","2":"2.042334e+14","3":"2.19"},{"1":"TORJAP","2":"5.511294e+14","3":"0.1373"},{"1":"TORJAP","2":"5.511294e+14","3":"0.1748"},{"1":"TORJAP","2":"3.103686e+14","3":"1.3899"},{"1":"TORMAX","2":"3.200000e+02","3":"1.4483"},{"1":"TORMAX","2":"5.600000e+02","3":"1.8853"},{"1":"TORMAX","2":"3.200000e+02","3":"1.1343"},{"1":"TORMAX","2":"1.600000e+02","3":"1.1512"},{"1":"TORMAX","2":"8.000000e+02","3":"5.5253"},{"1":"TORMAX","2":"4.800000e+02","3":"3.1542"},{"1":"TORMAX","2":"8.000000e+01","3":"0.2084"},{"1":"TORMAX","2":"3.200000e+02","3":"2.2014"},{"1":"TORMAX","2":"9.600000e+02","3":"4.3042"},{"1":"TORMAX","2":"8.800000e+02","3":"8.7397"},{"1":"TORMAX","2":"1.280000e+03","3":"7.2383"},{"1":"TORMAX","2":"2.400000e+02","3":"1.3229"},{"1":"TORMAX","2":"4.800000e+02","3":"1.8528"},{"1":"TORMAX","2":"4.000000e+02","3":"2.9635"},{"1":"TRIANG","2":"3.849552e+06","3":"0.6519"},{"1":"TRIANG","2":"1.347343e+07","3":"0.8889"},{"1":"TRIANG","2":"1.058627e+07","3":"0.7121"},{"1":"TRIANG","2":"1.828537e+07","3":"1.3168"},{"1":"TRIANG","2":"2.694686e+07","3":"2.1945"},{"1":"TRIANG","2":"5.774328e+06","3":"0.3389"},{"1":"TRIANG","2":"1.828537e+07","3":"1.866"},{"1":"TRIANG","2":"1.443582e+06","3":"1.2153"},{"1":"TRIANG","2":"4.811940e+05","3":"2.8673"},{"1":"TRIANG","2":"2.309731e+07","3":"1.5864"},{"1":"TRIANG","2":"1.347343e+07","3":"1.1809"},{"1":"TRIANG","2":"2.887164e+06","3":"0.1957"},{"1":"TRIANG","2":"1.154866e+07","3":"0.8642"},{"1":"TRIANG","2":"1.636060e+07","3":"1.0314"},{"1":"TRIANG","2":"1.443582e+06","3":"1.2785"},{"1":"TRIANG","2":"2.790925e+07","3":"1.8897"},{"1":"TRIANG","2":"1.251104e+07","3":"0.5774"},{"1":"TRIANG","2":"5.774328e+06","3":"0.5708"},{"1":"TRIANG","2":"1.347343e+07","3":"0.7933"},{"1":"TRIANG","2":"2.598448e+07","3":"1.9167"},{"1":"VERPER","2":"1.822800e+04","3":"0.0665"},{"1":"VERPER","2":"1.380120e+05","3":"0.68153"},{"1":"VERPER","2":"5.294800e+04","3":"0.1755"},{"1":"VERPER","2":"4.860800e+04","3":"0.3186"},{"1":"VERPER","2":"1.649200e+04","3":"0.0568"},{"1":"VERPER","2":"1.388800e+04","3":"0.0475"},{"1":"VERPER","2":"4.340000e+02","3":"0.1655"},{"1":"VERPER","2":"7.378000e+03","3":"0.30694"},{"1":"VERPER","2":"2.343600e+04","3":"0.0845"},{"1":"VERPER","2":"7.551600e+04","3":"0.2706"},{"1":"VERPER","2":"1.475600e+04","3":"0.0436"},{"1":"VERPER","2":"2.083200e+04","3":"0.0795"},{"1":"VERPER","2":"1.484280e+05","3":"0.6652"},{"1":"VERPER","2":"1.562400e+04","3":"0.0662"},{"1":"VERPER","2":"1.562400e+04","3":"0.0506"},{"1":"VICHYB","2":"1.152490e+05","3":"0.5658"},{"1":"VICHYB","2":"8.000000e+00","3":"0.5338"},{"1":"VICHYB","2":"1.352490e+05","3":"0.7828"},{"1":"VICHYB","2":"9.000000e+00","3":"0.7538"},{"1":"VICHYB","2":"7.000000e+00","3":"0.4347"},{"1":"VICHYB","2":"3.100000e+01","3":"2.2225"},{"1":"VICHYB","2":"3.557470e+05","3":"2.3745"},{"1":"VICHYB","2":"8.524900e+04","3":"0.2368"},{"1":"VICHYB","2":"1.100000e+01","3":"0.693"},{"1":"VICHYB","2":"6.000000e+00","3":"0.3571"},{"1":"VICHYB","2":"1.100000e+01","3":"0.6245"},{"1":"VICHYB","2":"9.000000e+00","3":"0.6065"},{"1":"VICHYB","2":"6.524900e+04","3":"0.2"},{"1":"VICHYB","2":"2.900000e+01","3":"1.5332"},{"1":"VICHYB","2":"2.652490e+05","3":"1.2056"},{"1":"VICSAT","2":"6.000000e+00","3":"0.0721"},{"1":"VICSAT","2":"2.700000e+01","3":"0.3702"},{"1":"VICSAT","2":"3.072000e+03","3":"0.2286"},{"1":"VICSAT","2":"3.272000e+03","3":"0.3512"},{"1":"VICSAT","2":"2.358000e+03","3":"0.1392"},{"1":"VICSAT","2":"5.516000e+03","3":"0.3732"},{"1":"VICSAT","2":"2.400000e+01","3":"0.2208"},{"1":"VICSAT","2":"2.086000e+03","3":"0.1915"},{"1":"VICSAT","2":"1.600600e+04","3":"1.2108"},{"1":"VICSAT","2":"6.072000e+03","3":"0.7572"},{"1":"VICSAT","2":"2.600000e+01","3":"0.4069"},{"1":"VICSAT","2":"5.986000e+03","3":"0.396"},{"1":"VICSAT","2":"2.944000e+03","3":"0.4301"},{"1":"VICSAT","2":"3.030000e+02","3":"0.1504"},{"1":"VICSAT","2":"3.700000e+01","3":"0.528"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>

Use relevant format for columns species Sp and biomass Vm:

```r
df$Sp <- as_factor(df$Sp)
df$Vm <- as.numeric(df$Vm)
```

Define numeric vector of species, for nested indexing in `Jags`:

```r
species <- as.numeric(df$Sp)
```

Define response variable, number of seeds:

```r
y <- log(df$NGrTotest)
```

Standardize explanatory variable, biomass

```r
x <- (df$Vm - mean(df$Vm))/sd(df$Vm)
```

Now build dataset:

```r
dat <- data.frame(y = y, x = x, species = species)
```

## Maximum-likelihood estimation

You can get maximum likelihood estimates for a linear regression of number of seeds on biomass with a species random effect on the intercept (partial pooling):

```r
mle <- lmer(y ~ x + (1 | species), data = dat)
summary(mle)
```

```
## Linear mixed model fit by REML ['lmerMod']
## Formula: y ~ x + (1 | species)
##    Data: dat
## 
## REML criterion at convergence: 2642.5
## 
## Scaled residuals: 
##     Min      1Q  Median      3Q     Max 
## -4.6761 -0.2704 -0.0120  0.2900  7.3163 
## 
## Random effects:
##  Groups   Name        Variance Std.Dev.
##  species  (Intercept) 113.146  10.637  
##  Residual               9.373   3.062  
## Number of obs: 488, groups:  species, 33
## 
## Fixed effects:
##             Estimate Std. Error t value
## (Intercept)  14.5258     1.8574   7.821
## x             0.4779     0.2421   1.974
## 
## Correlation of Fixed Effects:
##   (Intr)
## x 0.002
```

## Bayesian analysis with `Jags`

In `Jags`, you would use:

```r
# partial pooling model
partial_pooling <- function(){
  # likelihood
  for(i in 1:n){
    y[i] ~ dnorm(mu[i], tau.y)
    mu[i] <- a[species[i]] + b * x[i]
  }
  for (j in 1:nbspecies){
    a[j] ~ dnorm(mu.a, tau.a)
  }
  # priors
  tau.y <- 1 / (sigma.y * sigma.y)
  sigma.y ~ dunif(0,100)
  mu.a ~ dnorm(0,0.01)
  tau.a <- 1 / (sigma.a * sigma.a)
  sigma.a ~ dunif(0,100)
  b ~ dnorm(0,0.01)
}

# data
mydata <- list(y = y, 
               x = x, 
               n = length(y),
               species = species,
               nbspecies = length(levels(df$Sp)))

# initial values
inits <- function() list(mu.a = rnorm(1,0,5), 
                         sigma.a = runif(0,0,10), 
                         b = rnorm(1,0,5), 
                         sigma.y = runif(0,0,10))

# parameters to monitor
params <- c("mu.a", "sigma.a", "b", "sigma.y")

# call jags to fit model
bayes.jags <- jags(data = mydata,
                   inits = inits,
                   parameters.to.save = params,
                   model.file = partial_pooling,
                   n.chains = 2,
                   n.iter = 5000,
                   n.burnin = 1000,
                   n.thin = 1)
```

```
## Compiling model graph
##    Resolving undeclared variables
##    Allocating nodes
## Graph information:
##    Observed stochastic nodes: 488
##    Unobserved stochastic nodes: 37
##    Total graph size: 2484
## 
## Initializing model
```

```r
bayes.jags
```

```
## Inference for Bugs model at "/var/folders/r7/j0wqj1k95vz8w44sdxzm986c0000gn/T//RtmpP7MZ1A/modela81d7639dc1c.txt", fit using jags,
##  2 chains, each with 5000 iterations (first 1000 discarded)
##  n.sims = 8000 iterations saved
##           mu.vect sd.vect     2.5%      25%      50%      75%    97.5%  Rhat
## b           0.476   0.244   -0.005    0.314    0.472    0.638    0.964 1.001
## mu.a       14.025   1.916   10.146   12.774   14.009   15.299   17.760 1.001
## sigma.a    11.094   1.465    8.696   10.039   10.934   11.952   14.330 1.001
## sigma.y     3.072   0.101    2.884    3.002    3.069    3.138    3.277 1.001
## deviance 2478.206   8.704 2463.096 2472.022 2477.521 2483.650 2497.107 1.001
##          n.eff
## b         8000
## mu.a      8000
## sigma.a   8000
## sigma.y   8000
## deviance  8000
## 
## For each parameter, n.eff is a crude measure of effective sample size,
## and Rhat is the potential scale reduction factor (at convergence, Rhat=1).
## 
## DIC info (using the rule, pD = var(deviance)/2)
## pD = 37.9 and DIC = 2516.1
## DIC is an estimate of expected predictive error (lower deviance is better).
```

## Bayesian analysis with `brms`

In `brms`, you write:

```r
bayes.brms <- brm(y ~ x + (1 | species), 
                  data = dat,
                  chains = 2, # nb of chains
                  iter = 5000, # nb of iterations, including burnin
                  warmup = 1000, # burnin
                  thin = 1)
```

```
## Running /Library/Frameworks/R.framework/Resources/bin/R CMD SHLIB foo.c
## clang -mmacosx-version-min=10.13 -I"/Library/Frameworks/R.framework/Resources/include" -DNDEBUG   -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/Rcpp/include/"  -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/"  -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/unsupported"  -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/BH/include" -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/StanHeaders/include/src/"  -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/StanHeaders/include/"  -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppParallel/include/"  -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/rstan/include" -DEIGEN_NO_DEBUG  -DBOOST_DISABLE_ASSERTS  -DBOOST_PENDING_INTEGER_LOG2_HPP  -DSTAN_THREADS  -DBOOST_NO_AUTO_PTR  -include '/Library/Frameworks/R.framework/Versions/4.1/Resources/library/StanHeaders/include/stan/math/prim/mat/fun/Eigen.hpp'  -D_REENTRANT -DRCPP_PARALLEL_USE_TBB=1   -I/usr/local/include   -fPIC  -Wall -g -O2  -c foo.c -o foo.o
## In file included from <built-in>:1:
## In file included from /Library/Frameworks/R.framework/Versions/4.1/Resources/library/StanHeaders/include/stan/math/prim/mat/fun/Eigen.hpp:13:
## In file included from /Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/Eigen/Dense:1:
## In file included from /Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/Eigen/Core:88:
## /Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/Eigen/src/Core/util/Macros.h:628:1: error: unknown type name 'namespace'
## namespace Eigen {
## ^
## /Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/Eigen/src/Core/util/Macros.h:628:16: error: expected ';' after top level declarator
## namespace Eigen {
##                ^
##                ;
## In file included from <built-in>:1:
## In file included from /Library/Frameworks/R.framework/Versions/4.1/Resources/library/StanHeaders/include/stan/math/prim/mat/fun/Eigen.hpp:13:
## In file included from /Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/Eigen/Dense:1:
## /Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/Eigen/Core:96:10: fatal error: 'complex' file not found
## #include <complex>
##          ^~~~~~~~~
## 3 errors generated.
## make: *** [foo.o] Error 1
## 
## SAMPLING FOR MODEL 'c7dcb37f6d55803a5216866e78a4a476' NOW (CHAIN 1).
## Chain 1: 
## Chain 1: Gradient evaluation took 0.000102 seconds
## Chain 1: 1000 transitions using 10 leapfrog steps per transition would take 1.02 seconds.
## Chain 1: Adjust your expectations accordingly!
## Chain 1: 
## Chain 1: 
## Chain 1: Iteration:    1 / 5000 [  0%]  (Warmup)
## Chain 1: Iteration:  500 / 5000 [ 10%]  (Warmup)
## Chain 1: Iteration: 1000 / 5000 [ 20%]  (Warmup)
## Chain 1: Iteration: 1001 / 5000 [ 20%]  (Sampling)
## Chain 1: Iteration: 1500 / 5000 [ 30%]  (Sampling)
## Chain 1: Iteration: 2000 / 5000 [ 40%]  (Sampling)
## Chain 1: Iteration: 2500 / 5000 [ 50%]  (Sampling)
## Chain 1: Iteration: 3000 / 5000 [ 60%]  (Sampling)
## Chain 1: Iteration: 3500 / 5000 [ 70%]  (Sampling)
## Chain 1: Iteration: 4000 / 5000 [ 80%]  (Sampling)
## Chain 1: Iteration: 4500 / 5000 [ 90%]  (Sampling)
## Chain 1: Iteration: 5000 / 5000 [100%]  (Sampling)
## Chain 1: 
## Chain 1:  Elapsed Time: 0.656886 seconds (Warm-up)
## Chain 1:                3.37389 seconds (Sampling)
## Chain 1:                4.03077 seconds (Total)
## Chain 1: 
## 
## SAMPLING FOR MODEL 'c7dcb37f6d55803a5216866e78a4a476' NOW (CHAIN 2).
## Chain 2: 
## Chain 2: Gradient evaluation took 2.7e-05 seconds
## Chain 2: 1000 transitions using 10 leapfrog steps per transition would take 0.27 seconds.
## Chain 2: Adjust your expectations accordingly!
## Chain 2: 
## Chain 2: 
## Chain 2: Iteration:    1 / 5000 [  0%]  (Warmup)
## Chain 2: Iteration:  500 / 5000 [ 10%]  (Warmup)
## Chain 2: Iteration: 1000 / 5000 [ 20%]  (Warmup)
## Chain 2: Iteration: 1001 / 5000 [ 20%]  (Sampling)
## Chain 2: Iteration: 1500 / 5000 [ 30%]  (Sampling)
## Chain 2: Iteration: 2000 / 5000 [ 40%]  (Sampling)
## Chain 2: Iteration: 2500 / 5000 [ 50%]  (Sampling)
## Chain 2: Iteration: 3000 / 5000 [ 60%]  (Sampling)
## Chain 2: Iteration: 3500 / 5000 [ 70%]  (Sampling)
## Chain 2: Iteration: 4000 / 5000 [ 80%]  (Sampling)
## Chain 2: Iteration: 4500 / 5000 [ 90%]  (Sampling)
## Chain 2: Iteration: 5000 / 5000 [100%]  (Sampling)
## Chain 2: 
## Chain 2:  Elapsed Time: 0.593596 seconds (Warm-up)
## Chain 2:                2.34578 seconds (Sampling)
## Chain 2:                2.93937 seconds (Total)
## Chain 2:
```

Display the results:

```r
bayes.brms
```

```
##  Family: gaussian 
##   Links: mu = identity; sigma = identity 
## Formula: y ~ x + (1 | species) 
##    Data: dat (Number of observations: 488) 
##   Draws: 2 chains, each with iter = 5000; warmup = 1000; thin = 1;
##          total post-warmup draws = 8000
## 
## Group-Level Effects: 
## ~species (Number of levels: 33) 
##               Estimate Est.Error l-95% CI u-95% CI Rhat Bulk_ESS Tail_ESS
## sd(Intercept)    10.74      1.39     8.45    13.78 1.00      427      943
## 
## Population-Level Effects: 
##           Estimate Est.Error l-95% CI u-95% CI Rhat Bulk_ESS Tail_ESS
## Intercept    14.13      1.90    10.31    17.86 1.01      331      580
## x             0.47      0.25    -0.00     0.97 1.00     2317     3382
## 
## Family Specific Parameters: 
##       Estimate Est.Error l-95% CI u-95% CI Rhat Bulk_ESS Tail_ESS
## sigma     3.07      0.10     2.88     3.28 1.00     2667     3408
## 
## Draws were sampled using sampling(NUTS). For each parameter, Bulk_ESS
## and Tail_ESS are effective sample size measures, and Rhat is the potential
## scale reduction factor on split chains (at convergence, Rhat = 1).
```

You can extract a block of fixed effects:

```r
summary(bayes.brms)$fixed
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":[""],"name":["_rn_"],"type":[""],"align":["left"]},{"label":["Estimate"],"name":[1],"type":["dbl"],"align":["right"]},{"label":["Est.Error"],"name":[2],"type":["dbl"],"align":["right"]},{"label":["l-95% CI"],"name":[3],"type":["dbl"],"align":["right"]},{"label":["u-95% CI"],"name":[4],"type":["dbl"],"align":["right"]},{"label":["Rhat"],"name":[5],"type":["dbl"],"align":["right"]},{"label":["Bulk_ESS"],"name":[6],"type":["dbl"],"align":["right"]},{"label":["Tail_ESS"],"name":[7],"type":["dbl"],"align":["right"]}],"data":[{"1":"14.1340226","2":"1.9039258","3":"10.31129978","4":"17.864402","5":"1.005515","6":"330.6591","7":"580.1616","_rn_":"Intercept"},{"1":"0.4747192","2":"0.2477705","3":"-0.00471696","4":"0.968302","5":"1.000108","6":"2317.4168","7":"3382.0628","_rn_":"x"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>

And a block of random effects:

```r
summary(bayes.brms)$random
```

```
## $species
##               Estimate Est.Error l-95% CI u-95% CI     Rhat Bulk_ESS Tail_ESS
## sd(Intercept) 10.74273  1.388061 8.454543   13.776 1.001916 426.9568 943.1178
```

And visualize:

```r
plot(bayes.brms)
```

![](unnamed-chunk-34-1.png)<!-- -->

# GLMM with Poisson distribution

This example is from Jason Matthiopoulos' excellent book [*How to be a quantitative ecologist*](http://greenmaths.st-andrews.ac.uk/).
A survey of a coral reef uses 10 predefined linear transects covered by divers once every week. The response variable of interest is the abundance of a particular species of anemone as a function of water temperature. Counts of anemones are recorded at 20 regular line segments along the transect.

Let's simulate some data:

```r
set.seed(666)
transects <- 10
data <- NULL
for (tr in 1:transects){
  # random effect (intercept)
  ref <- rnorm(1,0,.5) 
  # water temperature gradient
  t <- runif(1, 18,22) + runif(1,-.2,0.2)*1:20 
  # Anemone gradient (expected response)
  ans <- exp(ref -14 + 1.8 * t - 0.045 * t^2) 
  # actual counts on 20 segments of the current transect
  an <- rpois(20, ans) 
  data <- rbind(data, cbind(rep(tr, 20), t, an))
}

data <- data.frame(Transect = data[,1],
                   Temperature = data[,2],
                   Anemones = data[,3])
```

Standardize temperature:

```r
boo <- data$Temperature
data$Temp <- (boo - mean(boo)) / sd(boo)
head(data)
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":[""],"name":["_rn_"],"type":[""],"align":["left"]},{"label":["Transect"],"name":[1],"type":["dbl"],"align":["right"]},{"label":["Temperature"],"name":[2],"type":["dbl"],"align":["right"]},{"label":["Anemones"],"name":[3],"type":["dbl"],"align":["right"]},{"label":["Temp"],"name":[4],"type":["dbl"],"align":["right"]}],"data":[{"1":"1","2":"21.79259","3":"65","4":"0.8852197","_rn_":"1"},{"1":"1","2":"21.67312","3":"70","4":"0.8201601","_rn_":"2"},{"1":"1","2":"21.55365","3":"65","4":"0.7551005","_rn_":"3"},{"1":"1","2":"21.43418","3":"61","4":"0.6900410","_rn_":"4"},{"1":"1","2":"21.31471","3":"81","4":"0.6249814","_rn_":"5"},{"1":"1","2":"21.19524","3":"85","4":"0.5599218","_rn_":"6"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>

## Maximum-likelihood estimation

With `lme4`, you write:

```r
fit_lme4 <- glmer(Anemones ~ Temp + I(Temp^2) + (1 | Transect), 
                  data = data, 
                  family = poisson)
summary(fit_lme4)
```

```
## Generalized linear mixed model fit by maximum likelihood (Laplace
##   Approximation) [glmerMod]
##  Family: poisson  ( log )
## Formula: Anemones ~ Temp + I(Temp^2) + (1 | Transect)
##    Data: data
## 
##      AIC      BIC   logLik deviance df.resid 
##   1380.2   1393.4   -686.1   1372.2      196 
## 
## Scaled residuals: 
##      Min       1Q   Median       3Q      Max 
## -2.32356 -0.66154 -0.03391  0.63964  2.67283 
## 
## Random effects:
##  Groups   Name        Variance Std.Dev.
##  Transect (Intercept) 0.1497   0.3869  
## Number of obs: 200, groups:  Transect, 10
## 
## Fixed effects:
##             Estimate Std. Error z value Pr(>|z|)    
## (Intercept)  3.95436    0.12368  31.971   <2e-16 ***
## Temp        -0.04608    0.02435  -1.892   0.0584 .  
## I(Temp^2)   -0.11122    0.01560  -7.130    1e-12 ***
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Correlation of Fixed Effects:
##           (Intr) Temp  
## Temp       0.037       
## I(Temp^2) -0.118 -0.294
```

## Bayesian analysis with `Jags`

In `Jags`, we fit the corresponding GLMM with:

```r
model <- function() {
  for (i in 1:n){
    count[i] ~ dpois(lambda[i])
    log(lambda[i]) <- a[transect[i]] + b[1] * x[i] + b[2] * pow(x[i],2)
  }
  for (j in 1:nbtransects){
    a[j] ~ dnorm (mu.a, tau.a)
  }
  mu.a ~ dnorm (0, 0.01)
  tau.a <- pow(sigma.a, -2)
  sigma.a ~ dunif (0, 100)
  b[1] ~ dnorm (0, 0.01)
  b[2] ~ dnorm (0, 0.01)
}

dat <- list(n = nrow(data), 
            nbtransects = transects, 
            x = data$Temp, 
            count = data$Anemones, 
            transect = data$Transect)

inits <- function() list(a = rnorm(transects), 
                         b = rnorm(2), 
                         mu.a = rnorm(1), 
                         sigma.a = runif(1))

par <- c ("a", "b", "mu.a", "sigma.a")

fit <- jags(data = dat, 
            inits = inits, 
            parameters.to.save = par, 
            model.file = model,
            n.chains = 2, 
            n.iter = 5000, 
            n.burn = 1000)
```

```
## Compiling model graph
##    Resolving undeclared variables
##    Allocating nodes
## Graph information:
##    Observed stochastic nodes: 200
##    Unobserved stochastic nodes: 14
##    Total graph size: 1622
## 
## Initializing model
```

```r
round(fit$BUGSoutput$summary[, -c(4,6)], 3)
```

```
##              mean    sd     2.5%      50%    97.5%  Rhat n.eff
## a[1]        4.383 0.026    4.332    4.384    4.435 1.001  2000
## a[2]        3.223 0.050    3.120    3.224    3.322 1.001  2000
## a[3]        3.584 0.058    3.471    3.584    3.696 1.001  2000
## a[4]        3.620 0.054    3.513    3.622    3.721 1.001  2000
## a[5]        4.092 0.050    3.990    4.092    4.190 1.001  2000
## a[6]        3.848 0.038    3.774    3.849    3.921 1.000  2000
## a[7]        4.476 0.029    4.419    4.475    4.531 1.002  1300
## a[8]        4.398 0.055    4.288    4.397    4.499 1.003   550
## a[9]        3.835 0.033    3.771    3.835    3.900 1.001  1700
## a[10]       4.076 0.029    4.018    4.076    4.132 1.001  2000
## b[1]       -0.047 0.024   -0.093   -0.047    0.000 1.001  2000
## b[2]       -0.111 0.016   -0.142   -0.111   -0.079 1.004   450
## deviance 1324.697 4.651 1317.432 1324.185 1335.440 1.001  2000
## mu.a        3.950 0.160    3.617    3.957    4.272 1.005   310
## sigma.a     0.482 0.140    0.296    0.457    0.826 1.003  1000
```

## Bayesian analysis with `brms`

What about with `brms`?

```r
bayes.brms <- brm(Anemones ~ Temp + I(Temp^2) + (1 | Transect), 
                  data = data,
                  family = poisson("log"),
                  chains = 2, # nb of chains
                  iter = 5000, # nb of iterations, including burnin
                  warmup = 1000, # burnin
                  thin = 1)
```

```
## Running /Library/Frameworks/R.framework/Resources/bin/R CMD SHLIB foo.c
## clang -mmacosx-version-min=10.13 -I"/Library/Frameworks/R.framework/Resources/include" -DNDEBUG   -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/Rcpp/include/"  -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/"  -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/unsupported"  -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/BH/include" -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/StanHeaders/include/src/"  -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/StanHeaders/include/"  -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppParallel/include/"  -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/rstan/include" -DEIGEN_NO_DEBUG  -DBOOST_DISABLE_ASSERTS  -DBOOST_PENDING_INTEGER_LOG2_HPP  -DSTAN_THREADS  -DBOOST_NO_AUTO_PTR  -include '/Library/Frameworks/R.framework/Versions/4.1/Resources/library/StanHeaders/include/stan/math/prim/mat/fun/Eigen.hpp'  -D_REENTRANT -DRCPP_PARALLEL_USE_TBB=1   -I/usr/local/include   -fPIC  -Wall -g -O2  -c foo.c -o foo.o
## In file included from <built-in>:1:
## In file included from /Library/Frameworks/R.framework/Versions/4.1/Resources/library/StanHeaders/include/stan/math/prim/mat/fun/Eigen.hpp:13:
## In file included from /Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/Eigen/Dense:1:
## In file included from /Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/Eigen/Core:88:
## /Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/Eigen/src/Core/util/Macros.h:628:1: error: unknown type name 'namespace'
## namespace Eigen {
## ^
## /Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/Eigen/src/Core/util/Macros.h:628:16: error: expected ';' after top level declarator
## namespace Eigen {
##                ^
##                ;
## In file included from <built-in>:1:
## In file included from /Library/Frameworks/R.framework/Versions/4.1/Resources/library/StanHeaders/include/stan/math/prim/mat/fun/Eigen.hpp:13:
## In file included from /Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/Eigen/Dense:1:
## /Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/Eigen/Core:96:10: fatal error: 'complex' file not found
## #include <complex>
##          ^~~~~~~~~
## 3 errors generated.
## make: *** [foo.o] Error 1
## 
## SAMPLING FOR MODEL '9326e0aa5a2dca4008c9e5b2ee4e86bf' NOW (CHAIN 1).
## Chain 1: 
## Chain 1: Gradient evaluation took 4.5e-05 seconds
## Chain 1: 1000 transitions using 10 leapfrog steps per transition would take 0.45 seconds.
## Chain 1: Adjust your expectations accordingly!
## Chain 1: 
## Chain 1: 
## Chain 1: Iteration:    1 / 5000 [  0%]  (Warmup)
## Chain 1: Iteration:  500 / 5000 [ 10%]  (Warmup)
## Chain 1: Iteration: 1000 / 5000 [ 20%]  (Warmup)
## Chain 1: Iteration: 1001 / 5000 [ 20%]  (Sampling)
## Chain 1: Iteration: 1500 / 5000 [ 30%]  (Sampling)
## Chain 1: Iteration: 2000 / 5000 [ 40%]  (Sampling)
## Chain 1: Iteration: 2500 / 5000 [ 50%]  (Sampling)
## Chain 1: Iteration: 3000 / 5000 [ 60%]  (Sampling)
## Chain 1: Iteration: 3500 / 5000 [ 70%]  (Sampling)
## Chain 1: Iteration: 4000 / 5000 [ 80%]  (Sampling)
## Chain 1: Iteration: 4500 / 5000 [ 90%]  (Sampling)
## Chain 1: Iteration: 5000 / 5000 [100%]  (Sampling)
## Chain 1: 
## Chain 1:  Elapsed Time: 0.893226 seconds (Warm-up)
## Chain 1:                4.91683 seconds (Sampling)
## Chain 1:                5.81006 seconds (Total)
## Chain 1: 
## 
## SAMPLING FOR MODEL '9326e0aa5a2dca4008c9e5b2ee4e86bf' NOW (CHAIN 2).
## Chain 2: 
## Chain 2: Gradient evaluation took 2.4e-05 seconds
## Chain 2: 1000 transitions using 10 leapfrog steps per transition would take 0.24 seconds.
## Chain 2: Adjust your expectations accordingly!
## Chain 2: 
## Chain 2: 
## Chain 2: Iteration:    1 / 5000 [  0%]  (Warmup)
## Chain 2: Iteration:  500 / 5000 [ 10%]  (Warmup)
## Chain 2: Iteration: 1000 / 5000 [ 20%]  (Warmup)
## Chain 2: Iteration: 1001 / 5000 [ 20%]  (Sampling)
## Chain 2: Iteration: 1500 / 5000 [ 30%]  (Sampling)
## Chain 2: Iteration: 2000 / 5000 [ 40%]  (Sampling)
## Chain 2: Iteration: 2500 / 5000 [ 50%]  (Sampling)
## Chain 2: Iteration: 3000 / 5000 [ 60%]  (Sampling)
## Chain 2: Iteration: 3500 / 5000 [ 70%]  (Sampling)
## Chain 2: Iteration: 4000 / 5000 [ 80%]  (Sampling)
## Chain 2: Iteration: 4500 / 5000 [ 90%]  (Sampling)
## Chain 2: Iteration: 5000 / 5000 [100%]  (Sampling)
## Chain 2: 
## Chain 2:  Elapsed Time: 1.21769 seconds (Warm-up)
## Chain 2:                4.45281 seconds (Sampling)
## Chain 2:                5.6705 seconds (Total)
## Chain 2:
```

Display results:

```r
bayes.brms
```

```
##  Family: poisson 
##   Links: mu = log 
## Formula: Anemones ~ Temp + I(Temp^2) + (1 | Transect) 
##    Data: data (Number of observations: 200) 
##   Draws: 2 chains, each with iter = 5000; warmup = 1000; thin = 1;
##          total post-warmup draws = 8000
## 
## Group-Level Effects: 
## ~Transect (Number of levels: 10) 
##               Estimate Est.Error l-95% CI u-95% CI Rhat Bulk_ESS Tail_ESS
## sd(Intercept)     0.47      0.14     0.29     0.81 1.00     1366     2089
## 
## Population-Level Effects: 
##           Estimate Est.Error l-95% CI u-95% CI Rhat Bulk_ESS Tail_ESS
## Intercept     3.95      0.16     3.63     4.26 1.00     1359     1751
## Temp         -0.05      0.02    -0.09     0.00 1.00     4126     4490
## ITempE2      -0.11      0.02    -0.14    -0.08 1.00     4018     4209
## 
## Draws were sampled using sampling(NUTS). For each parameter, Bulk_ESS
## and Tail_ESS are effective sample size measures, and Rhat is the potential
## scale reduction factor on split chains (at convergence, Rhat = 1).
```

Visualize:

```r
plot(bayes.brms)
```

![](unnamed-chunk-41-1.png)<!-- -->

We can assess the quality of fit of this model:

```r
pp_check(bayes.brms, ndraws = 100, type = 'ecdf_overlay')
```

![](unnamed-chunk-42-1.png)<!-- -->

As expected, the fit is almost perfect because we simulated data from this very model. 

What if we'd like to test the effect of temperature using WAIC? 

We fit a model with no effect of temperature:

```r
bayes.brms2 <- brm(Anemones ~ 1 + (1 | Transect), 
                  data = data,
                  family = poisson("log"),
                  chains = 2, # nb of chains
                  iter = 5000, # nb of iterations, including burnin
                  warmup = 1000, # burnin
                  thin = 1)
```

```
## Running /Library/Frameworks/R.framework/Resources/bin/R CMD SHLIB foo.c
## clang -mmacosx-version-min=10.13 -I"/Library/Frameworks/R.framework/Resources/include" -DNDEBUG   -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/Rcpp/include/"  -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/"  -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/unsupported"  -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/BH/include" -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/StanHeaders/include/src/"  -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/StanHeaders/include/"  -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppParallel/include/"  -I"/Library/Frameworks/R.framework/Versions/4.1/Resources/library/rstan/include" -DEIGEN_NO_DEBUG  -DBOOST_DISABLE_ASSERTS  -DBOOST_PENDING_INTEGER_LOG2_HPP  -DSTAN_THREADS  -DBOOST_NO_AUTO_PTR  -include '/Library/Frameworks/R.framework/Versions/4.1/Resources/library/StanHeaders/include/stan/math/prim/mat/fun/Eigen.hpp'  -D_REENTRANT -DRCPP_PARALLEL_USE_TBB=1   -I/usr/local/include   -fPIC  -Wall -g -O2  -c foo.c -o foo.o
## In file included from <built-in>:1:
## In file included from /Library/Frameworks/R.framework/Versions/4.1/Resources/library/StanHeaders/include/stan/math/prim/mat/fun/Eigen.hpp:13:
## In file included from /Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/Eigen/Dense:1:
## In file included from /Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/Eigen/Core:88:
## /Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/Eigen/src/Core/util/Macros.h:628:1: error: unknown type name 'namespace'
## namespace Eigen {
## ^
## /Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/Eigen/src/Core/util/Macros.h:628:16: error: expected ';' after top level declarator
## namespace Eigen {
##                ^
##                ;
## In file included from <built-in>:1:
## In file included from /Library/Frameworks/R.framework/Versions/4.1/Resources/library/StanHeaders/include/stan/math/prim/mat/fun/Eigen.hpp:13:
## In file included from /Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/Eigen/Dense:1:
## /Library/Frameworks/R.framework/Versions/4.1/Resources/library/RcppEigen/include/Eigen/Core:96:10: fatal error: 'complex' file not found
## #include <complex>
##          ^~~~~~~~~
## 3 errors generated.
## make: *** [foo.o] Error 1
## 
## SAMPLING FOR MODEL 'fd9804091450c044661531e89123d1dc' NOW (CHAIN 1).
## Chain 1: 
## Chain 1: Gradient evaluation took 4.4e-05 seconds
## Chain 1: 1000 transitions using 10 leapfrog steps per transition would take 0.44 seconds.
## Chain 1: Adjust your expectations accordingly!
## Chain 1: 
## Chain 1: 
## Chain 1: Iteration:    1 / 5000 [  0%]  (Warmup)
## Chain 1: Iteration:  500 / 5000 [ 10%]  (Warmup)
## Chain 1: Iteration: 1000 / 5000 [ 20%]  (Warmup)
## Chain 1: Iteration: 1001 / 5000 [ 20%]  (Sampling)
## Chain 1: Iteration: 1500 / 5000 [ 30%]  (Sampling)
## Chain 1: Iteration: 2000 / 5000 [ 40%]  (Sampling)
## Chain 1: Iteration: 2500 / 5000 [ 50%]  (Sampling)
## Chain 1: Iteration: 3000 / 5000 [ 60%]  (Sampling)
## Chain 1: Iteration: 3500 / 5000 [ 70%]  (Sampling)
## Chain 1: Iteration: 4000 / 5000 [ 80%]  (Sampling)
## Chain 1: Iteration: 4500 / 5000 [ 90%]  (Sampling)
## Chain 1: Iteration: 5000 / 5000 [100%]  (Sampling)
## Chain 1: 
## Chain 1:  Elapsed Time: 0.704799 seconds (Warm-up)
## Chain 1:                2.58736 seconds (Sampling)
## Chain 1:                3.29216 seconds (Total)
## Chain 1: 
## 
## SAMPLING FOR MODEL 'fd9804091450c044661531e89123d1dc' NOW (CHAIN 2).
## Chain 2: 
## Chain 2: Gradient evaluation took 1.8e-05 seconds
## Chain 2: 1000 transitions using 10 leapfrog steps per transition would take 0.18 seconds.
## Chain 2: Adjust your expectations accordingly!
## Chain 2: 
## Chain 2: 
## Chain 2: Iteration:    1 / 5000 [  0%]  (Warmup)
## Chain 2: Iteration:  500 / 5000 [ 10%]  (Warmup)
## Chain 2: Iteration: 1000 / 5000 [ 20%]  (Warmup)
## Chain 2: Iteration: 1001 / 5000 [ 20%]  (Sampling)
## Chain 2: Iteration: 1500 / 5000 [ 30%]  (Sampling)
## Chain 2: Iteration: 2000 / 5000 [ 40%]  (Sampling)
## Chain 2: Iteration: 2500 / 5000 [ 50%]  (Sampling)
## Chain 2: Iteration: 3000 / 5000 [ 60%]  (Sampling)
## Chain 2: Iteration: 3500 / 5000 [ 70%]  (Sampling)
## Chain 2: Iteration: 4000 / 5000 [ 80%]  (Sampling)
## Chain 2: Iteration: 4500 / 5000 [ 90%]  (Sampling)
## Chain 2: Iteration: 5000 / 5000 [100%]  (Sampling)
## Chain 2: 
## Chain 2:  Elapsed Time: 0.648008 seconds (Warm-up)
## Chain 2:                2.85026 seconds (Sampling)
## Chain 2:                3.49826 seconds (Total)
## Chain 2:
```

Then we compare both models, by ranking them with their WAIC or using ELPD:

```r
(waic1 <- waic(bayes.brms)) # waic model w/ tempterature
```

```
## 
## Computed from 8000 by 200 log-likelihood matrix
## 
##           Estimate   SE
## elpd_waic   -668.2  9.8
## p_waic        11.0  1.1
## waic        1336.3 19.7
```

```r
(waic2 <- waic(bayes.brms2)) # waic model wo/ tempterature
```

```
## 
## Computed from 8000 by 200 log-likelihood matrix
## 
##           Estimate   SE
## elpd_waic   -702.4 13.3
## p_waic        12.3  1.2
## waic        1404.9 26.5
## 
## 2 (1.0%) p_waic estimates greater than 0.4. We recommend trying loo instead.
```

```r
loo_compare(waic1, waic2)
```

```
##             elpd_diff se_diff
## bayes.brms    0.0       0.0  
## bayes.brms2 -34.3       8.9
```

## Conclusions

In all examples, you can check that i) there is little or no difference between `Jags` and `brms` numeric summaries of posterior distributions, and ii) maximum likelihood parameter estimates (`glm()`, `lmer()` and `glmer()` functions) and posterior means/medians are very close to each other. 

In contrast to Jags, Nimble or Stan, you do not need to code the likelihood yourself in `brms`, as long as it is available in the package. You can have a look to a list of distributions with `?brms::brmsfamily`, and there is also a possibility to define custom distributions with `custom_family()`. 

Note I haven't covered diagnostics of convergence, more [here](https://www.rensvandeschoot.com/brms-wambs/). You can also integrate `brms` outputs in a [tidy workflow with `tidybayes`](http://mjskay.github.io/tidybayes/articles/tidy-brms.html), and use the `ggplot()` magic.

As usual, the code is on GitHub, visit <https://github.com/oliviergimenez/fit-glmm-with-brms>.

Now I guess it's up to the students to pick their favorite Bayesian tool.  
