+++
date = "2017-08-06T12:00:00"
draft = false
tags = ["jags", "occupancy", "R", "hmm"]
title = "Fitting occupancy models in ADMB"
math = true
summary = """
"""

+++
 
<img style="float:left;margin-right:10px;margin-top:10px;margin-bottom:10px;" src="/img/admb.png">
Some time ago, a student of mine got stuck when fitting dynamic occupancy models to real data in Jags because of the computational burden. 

<!--more-->

We had a dataset with several thousands sites, more than 20 seasons and 4 surveys per season (yeah!). 

We thought of using Unmarked instead (the likelihood is written in C++ and used through Rcpp), but dynamic models with false positives and/or random effects are not (yet?) implemented, and we were interested in considering both in our analysis. Some years ago, I had the opportunity to learn ADMB in a NCEAS meeting (thanks Hans Skaug!), I thought I would give it a try. ADMB allows you to write down any likelihood functions yourself and to incorporate random effects in an efficient way. It's known to be fast for reasons I won't go into here. Last but not least, ADMB can be run from R like JAGS and Unmarked (thanks Ben Bolker!).

Here we go. I first simulate some data, then fit a dynamic model using ADMB, JAGS and Unmarked and finally perform a quick benchmarking. I'm going for a standard dynamic model, because the aims are i) to verify that JAGS is slower than Unmarked, ii) that ADMB is closer to Unmarked than JAGS in terms of time computation. If ii) is verified, then it will be worth the effort coding everything in ADMB.

The results are available on RPub [here](http://rpubs.com/ogimenez/297167). The code is available on GitHub [here](https://github.com/oliviergimenez/occupancy_in_ADMB). 

Hope this is useful.
