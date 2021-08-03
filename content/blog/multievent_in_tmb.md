+++
date = "2017-08-21T12:00:00"
draft = false
tags = ["capture-recapture", "multievent", "R", "hmm", "TMB"]
title = "Fitting HMM/multievent capture-recapture models with TMB"
math = true
summary = """
"""

+++
 
<img style="float:left;margin-right:10px;margin-top:10px;" src="/img/tmb.png">
Following my attempts to fit a HMM model to [capture-recapture data with Rcpp](http://localhost:1313/post/multievent_in_rcpp/) and to [occupancy data with ADMB](http://localhost:1313/post/occupancy_in_admb/), a few colleagues suggested TMB as a potential alternative for several reasons (fast, allows for parallel computations, works with R, accomodates spatial stuff, easy implementation of random effects, and probably other reasons that I don't know).

<!--more-->

I found materials on the internet to teach myself TMB, at least what I needed to implement a simple HMM model. See [here](http://seananderson.ca/2014/10/17/tmb.html) for a linear regression and a Gompertz state space model examples, [here](https://www.youtube.com/watch?v=A5CLrhzNzVU) for the same linear regression example on Youtube (that's awesome!) and many other examples [here](http://kaskr.github.io/adcomp/examples.html). However, I got stuck and posted my desperate request for help on the [TMB forum](https://groups.google.com/forum/#!forum/tmb-users). Guess what, I got an answer less than a few hours after - thank you Mollie Brooks!

The R code is available on my GitHub [here](https://github.com/oliviergimenez/hmm_tmb). Minimizing the deviance coded with TMB was... wait for it... > 300 times faster than using the deviance coded in standard R.

Hope this is useful.
