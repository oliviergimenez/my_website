+++
date = "2018-01-13"
draft = false
tags = ["bugs", "R", "rstats", "OpenBUGS"]
title = "Run OpenBUGS on a Mac"
math = true
summary = """
"""

+++
 
<img style="float:left;margin-right:10px;margin-top:10px;width:200px;margin-bottom:5px" src="/img/bugs.jpg"> 
I had to use the good old `OpenBUGS` for some analyses that cannot be done in `JAGS`. Below are the steps to install `OpenBUGS` then to run it from your Mac either natively or from `R`. This tutorial is an adaptation of [this post](https://sites.google.com/site/mmeclimate/-bayesmet/openbugs-on-mac-os-x) and [that one](http://www.davideagle.org/r-2/bayesian-modeling-using-winbugs-and-openbugs/running-openbugs-on-mac-using-wine). 

<!--more-->

1. If not done already, install [Homebrew](https://brew.sh/). This program will make the installation of any other programs on your Mac so easy!

2. Install [Wine](https://www.winehq.org/) which will allow you to run any Windows programs (.exe) on your Mac. To do so, start by [opening Terminal](http://blog.teamtreehouse.com/introduction-to-the-mac-os-x-command-line), then type in the command: *brew install wine*
    
3. Next, download the Windows version of `OpenBUGS` [here](https://www.mrc-bsu.cam.ac.uk/training/short-courses/bayescourse/download/)

4. To install `OpenBUGS`, still in Terminal, go to the directory where the file was downloaded and type (you might need to unzip the file you downloaded first): *wine OpenBUGS323setup.exe*
    
5. `OpenBUGS` is now installed and ready to be used! You can run it by first going to the directory where `OpenBUGS` was installed. On my laptop, it can be achieved via the command: *cd /Applications/OpenBUGS323*

6. Then, you just need to tye in the following command in the Terminal, and you should see an OpenBUGS windows poping up: *wine OpenBUGS*

Now we would like to run `OpenBUGS` from `R`.

7. Install the package `R2OpenBUGS` by typing in the `R` console:

```r
if(!require(R2OpenBUGS)) install.packages("R2OpenBUGS")
```

```
## Loading required package: R2OpenBUGS
```

8. Now let's see whether everything works well by running the classical `BUGS` `school` example:

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
schools.sim <- bugs(data, inits, model.file = model.file1,parameters=parameters,n.chains = 3, n.iter = 1000, OpenBUGS.pgm=OpenBUGS.pgm, WINE=WINE, WINEPATH=WINEPATH,useWINE=T)
```

`R` will pause. You might get a weird message starting by err:ole, just ignore it. When the run is complete, a prompt will reappear, then just type the following command to get the result:


```r
print(schools.sim)
```

```
## Inference for Bugs model at "/Users/oliviergimenez/Desktop/nummodel.txt", 
## Current: 3 chains, each with 1000 iterations (first 500 discarded)
## Cumulative: n.sims = 1500 iterations saved
##             mean  sd 2.5%  25%  50%  75% 97.5% Rhat n.eff
## theta[1]    12.2 7.9 -1.3  7.5 11.2 16.4  32.1  1.0    62
## theta[2]     9.1 6.5 -4.0  5.1  9.4 13.2  21.4  1.0   150
## theta[3]     7.8 7.7 -9.4  3.6  8.5 12.6  21.1  1.0   360
## theta[4]     8.8 6.6 -4.5  4.5  9.2 13.3  20.4  1.0   110
## theta[5]     6.8 6.9 -8.2  2.3  7.5 11.4  17.7  1.0   410
## theta[6]     7.3 7.2 -8.6  2.7  8.2 11.8  18.9  1.0   190
## theta[7]    11.5 6.4 -0.3  7.5 11.2 15.7  25.0  1.1    42
## theta[8]     9.7 7.6 -4.7  5.1  9.6 14.4  25.1  1.0   130
## mu.theta     9.2 5.2 -1.2  5.8  9.3 12.5  18.2  1.0    88
## sigma.theta  5.9 5.6  0.2  1.7  4.4  8.5  20.2  1.1    51
## deviance    60.7 2.2 57.2 59.2 60.1 61.9  65.6  1.0   120
## 
## For each parameter, n.eff is a crude measure of effective sample size,
## and Rhat is the potential scale reduction factor (at convergence, Rhat=1).
## 
## DIC info (using the rule, pD = Dbar-Dhat)
## pD = 2.8 and DIC = 63.4
## DIC is an estimate of expected predictive error (lower deviance is better).
```

When run natively, `WinBUGS` and `OpenBUGS` have nice debugging capabilities; also, you can see what is going on, I mean the program reading the data, generating inits, and so on. To get the `OpenBUGS` window with a bunch of useful info, just add `debug=T` to the call of the `bugs` function, and re-run the model


```r
schools.sim <- bugs(data, inits, model.file = model.file1,parameters=parameters,n.chains = 3, n.iter = 1000, OpenBUGS.pgm=OpenBUGS.pgm, WINE=WINE, WINEPATH=WINEPATH,useWINE=T,debug=T)
```

```
## arguments 'show.output.on.console', 'minimized' and 'invisible' are for Windows only
```

You will have to close the `OpenBUGS` window to get the prompt back.

