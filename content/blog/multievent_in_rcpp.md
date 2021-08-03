+++
date = "2017-08-11T12:00:00"
draft = false
tags = ["capture-recapture", "multievent", "R", "hmm"]
title = "Fitting multievent capture-recapture models with Rcpp"
math = true
summary = """
"""

+++
 
<img style="float:left;margin-right:10px;margin-top:10px;" src="/img/seamless.png">
Following my previous post on [using ADMB to fit hidden Markov models](https://oliviergimenez.github.io/post/occupancy_in_admb/), I took some time to learn how to use Rcpp ([Eddelbuettel & Francois 2011](https://www.jstatsoft.org/article/view/v040i08); [Eddelbuettel 2013](http://www.springer.com/us/book/9781461468677)), a package that gives friendly access to the power of C++ and increase the speed of your R programs. Kudos to Dirk Eddelbuettel, Romain Francois and their colleagues, Rcpp is awesome! 

<!--more-->

I started with the excellent [Rcpp chapter](http://adv-r.had.co.nz/Rcpp.html) in the [Advanced R](http://adv-r.had.co.nz/) book by Hadley Wickham which I complemented with the various [vignettes](https://cran.r-project.org/web/packages/Rcpp/index.html) that come with the package. As always, I googled the problems I had and often ended up finding the solution on [stackoverflow](https://stackoverflow.com/). The [rcpp-devel discussion list](http://lists.r-forge.r-project.org/mailman/listinfo/rcpp-devel) is the place where questions should be asked about Rcpp.

My objective was to implement the likelihood of a relatively simple multievent capture-recapture model ([Pradel 2005](http://onlinelibrary.wiley.com/doi/10.1111/j.1541-0420.2005.00318.x/abstract)) with Rcpp. I recycled some R code I had and a dataset on shearwaters I used in a paper ([Gimenez et al. 2012](https://dl.dropboxusercontent.com/u/23160641/my-pubs/Gimenezetal2012TPB.pdf)).

The R code is available on my GitHub [here](https://github.com/oliviergimenez/multieventRcpp). To run it, you just need to type Rcpp::sourceCpp('multi event.cpp') in the console. I'm convinced that the code can be improved, but this simple exercise showed that minimizing the deviance coded with Rcpp and calculating the Hessian was 10 times faster than using the deviance coded in standard R.

Next steps will be to go for RcppArmadillo for matrix computations and RcppNumerical for optimisation (and numerical integration for random effects).

Hope this is useful.

**Update:** Following an advice from Romain Francois and Dirk Eddelbuettel (the Rcpp gurus), I have switched to [RcppArmadillo](https://cran.r-project.org/web/packages/RcppArmadillo/index.html) to rely on the code developed by professionals and decades of testing. Now the RcppArmadillo code is 50 times faster than basic R! The code is available on [my GitHub](https://github.com/oliviergimenez/multieventRcpp).