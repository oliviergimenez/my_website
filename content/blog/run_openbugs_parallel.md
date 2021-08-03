+++
date = "2018-01-14"
draft = false
tags = ["bugs", "R", "rstats", "OpenBUGS", "parallel computation"]
title = "Running OpenBUGS in parallel"
math = true
summary = """
"""

+++
 
<img style="float:left;margin-right:10px;margin-top:10px;width:200px;margin-bottom:5px" src="/img/parcomp.jpg"> 
Recently, I have been using `OpenBUGS` for some analyses that `JAGS` cannot do. However, `JAGS` can be run in parallel through [the `jagsUI` package](https://github.com/kenkellner/jagsUI), which can save you some precious time. So the question is how to run several chains in parallel with `OpenBUGS`. 

<!--more-->

Well, first you'll need to install `OpenBUGS` (if you're on a Mac, check out [this short tutorial](https://oliviergimenez.github.io/post/run_openbugs_on_mac/)). Then, you'll need to run `OpenBUGS` from `R` through the pacage `R2OpenBUGS`, which you can install via:

```r
if(!require(R2OpenBUGS)) install.packages("R2OpenBUGS")
```

```
## Loading required package: R2OpenBUGS
```

## Standard analysis

Now let's run the classical `BUGS` `school` example:

Load the `OpenBUGS` Package

```r
library(R2OpenBUGS)
```

Load the data

```r
data(schools)
```

Define the model, write it to a text file and have a look

```r
nummodel <- function(){
for (j in 1:J){
  y[j] ~ dnorm (theta[j], tau.y[j])
  theta[j] ~ dnorm (mu.theta, tau.theta)
  tau.y[j] <- pow(sigma.y[j], -2)}
mu.theta ~ dnorm (0.0, 1.0E-6)
tau.theta <- pow(sigma.theta, -2)
sigma.theta ~ dunif (0, 1000)
}
write.model(nummodel, "nummodel.txt")
model.file1 = paste(getwd(),"nummodel.txt", sep="/")
file.show("nummodel.txt")
```

Prepare the data for input into OpenBUGS

```r
J <- nrow(schools)
y <- schools$estimate
sigma.y <- schools$sd
data <- list ("J", "y", "sigma.y")
```

Initialization of variables

```r
inits <- function(){
  list(theta = rnorm(J, 0, 100), mu.theta = rnorm(1, 0, 100), sigma.theta = runif(1, 0, 100))}
```

Set the `Wine` working directory and the directory to `OpenBUGS`, and change the OpenBUGS.exe location as necessary:

```r
WINE="/usr/local/Cellar/wine/2.0.4/bin/wine"
WINEPATH="/usr/local/Cellar/wine/2.0.4/bin/winepath"
OpenBUGS.pgm="/Applications/OpenBUGS323/OpenBUGS.exe"
```

The are the parameters to save

```r
parameters = c("theta", "mu.theta", "sigma.theta")
```

Run the model

```r
ptm <- proc.time()
schools.sim <- bugs(data, inits, model.file = model.file1,parameters=parameters,n.chains = 2, n.iter = 500000, n.burnin = 10000, OpenBUGS.pgm=OpenBUGS.pgm, WINE=WINE, WINEPATH=WINEPATH,useWINE=T)
elapsed_time <- proc.time() - ptm
elapsed_time 
```

```
##    user  system elapsed 
##  50.835   2.053  55.010
```


```r
print(schools.sim)
```

```
## Inference for Bugs model at "/Users/oliviergimenez/Desktop/nummodel.txt", 
## Current: 2 chains, each with 5e+05 iterations (first 10000 discarded)
## Cumulative: n.sims = 980000 iterations saved
##             mean  sd  2.5%  25%  50%  75% 97.5% Rhat  n.eff
## theta[1]    11.6 8.4  -1.9  6.1 10.5 15.8  32.0    1 370000
## theta[2]     8.0 6.4  -4.8  4.0  8.0 12.0  20.9    1  67000
## theta[3]     6.4 7.8 -11.3  2.2  6.8 11.2  20.9    1  55000
## theta[4]     7.7 6.6  -5.7  3.7  7.8 11.8  20.9    1  70000
## theta[5]     5.5 6.5  -8.8  1.6  5.9  9.8  17.1    1  26000
## theta[6]     6.2 6.9  -8.9  2.3  6.6 10.7  18.9    1  23000
## theta[7]    10.7 6.9  -1.4  6.0 10.1 14.7  26.2    1 480000
## theta[8]     8.7 7.9  -6.8  4.0  8.4 13.0  25.7    1  76000
## mu.theta     8.1 5.3  -2.0  4.7  8.1 11.4  18.5    1  30000
## sigma.theta  6.6 5.7   0.2  2.5  5.2  9.1  20.9    1  12000
## deviance    60.5 2.2  57.0 59.1 60.1 61.4  66.0    1 980000
## 
## For each parameter, n.eff is a crude measure of effective sample size,
## and Rhat is the potential scale reduction factor (at convergence, Rhat=1).
## 
## DIC info (using the rule, pD = Dbar-Dhat)
## pD = 2.8 and DIC = 63.2
## DIC is an estimate of expected predictive error (lower deviance is better).
```

## Parallel computations

To run several chains in parallel, we'll follow the steps described in [this nice post](http://www.petrkeil.com/?p=63).


```r
# loading packages
library(snow)
library(snowfall)

# setting the number of CPUs to be 2
sfInit(parallel=TRUE, cpus=2)
```

```
## Warning in searchCommandline(parallel, cpus = cpus, type
## = type, socketHosts = socketHosts, : Unknown option on
## commandline: rmarkdown::render('/Users/oliviergimenez/Desktop/
## run_openbugs_in_parallel.Rmd',~+~~+~encoding~+~
```

```
## R Version:  R version 3.4.3 (2017-11-30)
```

```
## snowfall 1.84-6.1 initialized (using snow 0.4-2): parallel execution on 2 CPUs.
```

```r
# and assigning the R2OpenBUGS library to each CPU
sfLibrary(R2OpenBUGS)
```

```
## Library R2OpenBUGS loaded.
```

```
## Library R2OpenBUGS loaded in cluster.
```

```r
# create list of data
J <- nrow(schools)
y <- schools$estimate
sigma.y <- schools$sd
x.data <- list (J=J, y=y, sigma.y=sigma.y)

# creating separate directory for each CPU process
folder1 <- paste(getwd(), "/chain1", sep="")
folder2 <- paste(getwd(), "/chain2", sep="")
dir.create(folder1); dir.create(folder2); 
 
# sinking the model into a file in each directory
for (folder in c(folder1, folder2))
{
  sink(paste(folder, "/nummodel.txt", sep=""))
cat("
	model{
for (j in 1:J){
  y[j] ~ dnorm (theta[j], tau.y[j])
  theta[j] ~ dnorm (mu.theta, tau.theta)
  tau.y[j] <- pow(sigma.y[j], -2)}
mu.theta ~ dnorm (0.0, 1.0E-6)
tau.theta <- pow(sigma.theta, -2)
sigma.theta ~ dunif (0, 1000)
	}
")
  sink()
}
 
# defining the function that will run MCMC on each CPU
# Arguments:
# chain - will be 1 or 2
# x.data - the data list
# params - parameters to be monitored
parallel.bugs <- function(chain, x.data, params)
{
  # a. defining directory for each CPU
  sub.folder <- paste(getwd(),"/chain", chain, sep="")
 
  # b. specifying the initial MCMC values
  inits <- function()list(theta = rnorm(x.data$J, 0, 100), mu.theta = rnorm(1, 0, 100), sigma.theta = runif(1, 0, 100))
 
  # c. calling OpenBugs
  # (you may need to change the OpenBUGS.pgm directory)
  # je suis sous Mac, je fais tourner OpenBUGS via Wine
  bugs(data=x.data, inits=inits, parameters.to.save=params,
             n.iter = 500000, n.burnin = 10000, n.chains=1,
             model.file="nummodel.txt", debug=FALSE, codaPkg=TRUE,
             useWINE=TRUE, OpenBUGS.pgm = "/Applications/OpenBUGS323/OpenBUGS.exe",
             working.directory = sub.folder,
             WINE="/usr/local/Cellar/wine/2.0.4/bin/wine", 
             WINEPATH="/usr/local/Cellar/wine/2.0.4/bin/winepath")
}
 
# setting the parameters to be monitored
params <- c("theta", "mu.theta", "sigma.theta")
 
# calling the sfLapply function that will run
# parallel.bugs on each of the 2 CPUs
ptm <- proc.time()
sfLapply(1:2, fun=parallel.bugs, x.data=x.data, params=params)
```

```
## [[1]]
## [1] "/Users/oliviergimenez/Desktop/chain1/CODAchain1.txt"
## 
## [[2]]
## [1] "/Users/oliviergimenez/Desktop/chain2/CODAchain1.txt"
```

```r
elapsed_time = proc.time() - ptm
elapsed_time
```

```
##    user  system elapsed 
##   0.013   0.000  32.157
```

```r
# locating position of each CODA chain
chain1 <- paste(folder1, "/CODAchain1.txt", sep="")
chain2 <- paste(folder2, "/CODAchain1.txt", sep="")
 
# and, finally, getting the results
res <- read.bugs(c(chain1, chain2))
```

```
## Abstracting deviance ... 490000 valid values
## Abstracting mu.theta ... 490000 valid values
## Abstracting sigma.theta ... 490000 valid values
## Abstracting theta[1] ... 490000 valid values
## Abstracting theta[2] ... 490000 valid values
## Abstracting theta[3] ... 490000 valid values
## Abstracting theta[4] ... 490000 valid values
## Abstracting theta[5] ... 490000 valid values
## Abstracting theta[6] ... 490000 valid values
## Abstracting theta[7] ... 490000 valid values
## Abstracting theta[8] ... 490000 valid values
## Abstracting deviance ... 490000 valid values
## Abstracting mu.theta ... 490000 valid values
## Abstracting sigma.theta ... 490000 valid values
## Abstracting theta[1] ... 490000 valid values
## Abstracting theta[2] ... 490000 valid values
## Abstracting theta[3] ... 490000 valid values
## Abstracting theta[4] ... 490000 valid values
## Abstracting theta[5] ... 490000 valid values
## Abstracting theta[6] ... 490000 valid values
## Abstracting theta[7] ... 490000 valid values
## Abstracting theta[8] ... 490000 valid values
```

```r
summary(res)
```

```
## 
## Iterations = 10001:5e+05
## Thinning interval = 1 
## Number of chains = 2 
## Sample size per chain = 490000 
## 
## 1. Empirical mean and standard deviation for each variable,
##    plus standard error of the mean:
## 
##               Mean    SD Naive SE Time-series SE
## deviance    60.453 2.221 0.002243       0.005737
## mu.theta     8.109 5.261 0.005315       0.020596
## sigma.theta  6.610 5.682 0.005740       0.027336
## theta[1]    11.697 8.407 0.008493       0.027974
## theta[2]     8.023 6.395 0.006460       0.019264
## theta[3]     6.365 7.866 0.007946       0.022485
## theta[4]     7.735 6.601 0.006668       0.019766
## theta[5]     5.467 6.504 0.006570       0.022580
## theta[6]     6.234 6.885 0.006955       0.021332
## theta[7]    10.727 6.891 0.006961       0.023304
## theta[8]     8.648 7.892 0.007972       0.021541
## 
## 2. Quantiles for each variable:
## 
##                 2.5%    25%    50%    75% 97.5%
## deviance     57.0200 59.120 60.040 61.430 65.99
## mu.theta     -2.0600  4.784  8.066 11.410 18.50
## sigma.theta   0.2275  2.456  5.275  9.190 20.82
## theta[1]     -1.8850  6.195 10.560 15.880 32.04
## theta[2]     -4.8350  4.049  8.004 11.980 20.96
## theta[3]    -11.4800  2.194  6.871 11.170 20.90
## theta[4]     -5.7500  3.711  7.784 11.820 20.92
## theta[5]     -8.8490  1.603  5.938  9.834 17.14
## theta[6]     -8.8940  2.255  6.632 10.680 18.95
## theta[7]     -1.3450  6.125 10.140 14.680 26.27
## theta[8]     -6.8910  4.037  8.409 12.960 25.72
```


