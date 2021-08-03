+++
date = "2019-01-04"
draft = false
tags = ["statistical ecology", "estimation", "delta-method", "rstats"]
title = "Calculate the standard error of any function using the delta method"
math = true
summary = """
"""

+++

In statistical ecology, we often need to calculate the sampling variance of a function of an estimate of which we do know the sampling variance. I keep forgetting how to implement the so-called delta method in `R` that allows to get an approximation of this quantity. So in this post I go through two examples in population ecology that should help me remembering. I use the `deltamethod` function from the [`msm` package](https://cran.r-project.org/web/packages/msm/index.html). 

<!--more-->

Load the package `msm` and get some help on the delta-method function:

```r
library(msm)
?deltamethod
```

Further examples can be obtained by typing in:

```r
example(deltamethod)
```

For a nice introduction to the delta method, check [that](http://www.phidot.org/software/mark/docs/book/pdf/app_2.pdf) out. Two papers are worth reading on the topic: ['Who Invented the Delta Method?'](https://www.tandfonline.com/doi/abs/10.1080/00031305.2012.687494) by Jay M. Ver Hoef and ['Approximating variance of demographic parameters using the delta method: A reference for avian biologists'](https://bioone.org/journals/The-Condor/volume-109/issue-4/0010-5422(2007)109[949:AVODPU]2.0.CO;2/APPROXIMATING-VARIANCE-OF-DEMOGRAPHIC-PARAMETERS-USING-THE-DELTA-METHOD/10.1650/0010-5422(2007)109[949:AVODPU]2.0.CO;2.full) by Larkin A. Powell.

### Simple example

A simple example is when, for example, you get $\phi$ (ignore the traditional hat) an estimate of a survival probability on the logit scale in some capture-recapture analyses, and you would like to get the standard error (SE) of survival on its natural scale. 

For example, say $\text{logit}(\phi) = \text{lphi} = -0.4473122$ with $\text{SE} = 0.3362757$. 

To obtain $\phi$, you back-transform $\text{lphi}$ using the reciprocal function of the logit function: $$\phi = \displaystyle{\frac{\exp(\text{lphi})}{1+\exp(\text{lphi})}} =  \displaystyle{\frac{1}{1+\exp(\text{-lphi})}} = \displaystyle{\frac{1}{1+\exp(\text{-(-0.4473122)})}} = 0.39.$$

What about the SE of $\phi$? Well, a direct application of the `deltamethod` function from the `msm` package gives the answer:

```r
deltamethod(~ 1/(1+exp(-x1)), -0.4473122, 0.3362757^2)
```

```
## [1] 0.07999999
```

Two things to take care of: 

* First, the variables in the formula must be labelled $x_1, x_2, \text{text}$. You cannot use $x, y, z, ...$ for example. Just numbered $x$'s.

* Second, the input parameters are the estimate and its squared SE (not the SE), and by default you will get as an output the SE (not the squared SE) of the function defined by the formula. 

## Complex example

This example deals with an occupancy model. It is a bit more complex than the previous example because we consider a function of several parameters for which we would like to calculate its SE. I assume that occupancy at first occasion was estimated along with its SE, and that one would like to obtain the SE of subsequent occupancy probabilities.

I calculate time-dependent occupancy probabilities with the following formula $$\psi_{t+1} = \psi_t (1 - \varepsilon) + (1 - \psi_t) \gamma$$ where $\varepsilon$ is extinction, $\gamma$ is colonisation and $\psi_t$ is occupancy year $t$.

We assume that we obtained the following parameter estimates:

```r
epsilon = 0.39
gamma = 0.07
psi_init = 0.1 # first-occasion occupancy
```

with corresponding SEs:

```r
se_epsilon = 0.08
se_psi_init = 0.01
se_gamma = 0.05
```

We will estimate occupancy and get SEs at 10 occasions, which we store in two matrices (column vectors):

```r
psi = matrix(0, nrow = 10, ncol = 1)
psi_se = matrix(0, nrow = 10, ncol = 1)
```

The first element is occupancy at first occasion:

```r
psi[1,] <- psi_init
psi_se[1,] <- se_psi_init
```

Then we iterate calculations using the formula above:

```r
for(i in 2:10){
	psi_current <- psi[i-1,]
	psi_se_current <- psi_se[i-1,]
	estmean <- c(psi_current,epsilon,gamma)
	estvar <- diag(c(psi_se_current,se_epsilon,se_gamma)^2)
	psi[i,] = (psi_current*(1-epsilon)) + ((1-psi_current)*gamma) # recurrence formula
	psi_se[i,] = deltamethod(~ x1*(1-x2) + (1-x1)*x3, estmean, estvar) # delta-method
}
```

Display results:

```r
data.frame(psi = psi,sterr_psi = psi_se)
```

```
##          psi  sterr_psi
## 1  0.1000000 0.01000000
## 2  0.1240000 0.04602347
## 3  0.1369600 0.05132740
## 4  0.1439584 0.05244394
## 5  0.1477375 0.05259904
## 6  0.1497783 0.05255782
## 7  0.1508803 0.05250010
## 8  0.1514753 0.05245886
## 9  0.1517967 0.05243372
## 10 0.1519702 0.05241934
```

Here, we assumed that sampling correlation was 0, in other words that the estimates of $\psi$, $\gamma$ and $\epsilon$ were independent, hence the use of a diagonal matrix for `estvar`. It is possible to use a non-diagonal covariance matrix to account for non-null correlation.
