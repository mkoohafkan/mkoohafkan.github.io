---
layout:     post
title:      BSTEM and SED-WAT at SEDHYD 2019
date:       2019-07-16 20:00:00
summary:    I'm an author on two papers at the 2019 Federal Interagency Sedimentation and Hydrologic Modeling Conference. Check them out!
categories: horntooter
commentIssueId: 41
color: darken-red
---

The 
[Federal Interagency Sedimentation and Hydrologic Modeling 
Conference](https://www.sedhyd.org/2019) 
(SEDHYD) has come and gone, held as usual in Reno, NV. 
While I wasn't able to attend, I did submit two papers
to the conference. The first paper is all about
[using HEC-WAT and HEC-RAS-Sediment to evaluate the effect of hydrologic uncertainty on bed evolution](/docs/2019-sedwat-237).
I posted about this project [a little while ago]({% post_url 2018-09-29-plotting-marginal-distributions-anywhere-with-ggplot %}),
but I focused on the R code used to generate some of the figures and 
didn't go into much detail on how the data for these figures was
generated. This short paper goes into more detail about how HEC-WAT
and HEC-RAS can be used together to quantify the effect of 
two main sources of morphological uncertainty---flow magnitude and 
timing---on flood risk. If you're interested in doing uncertainty 
modeling with HEC-RAS, this paper is a good introduction to tools 
provided by HEC.

The second paper focuses on 
[modeling bank migration on the Missouri River with HEC-RAS](/docs/2019-sedwat-232).
I spent a lot of time working on the Bank Stability and Toe Erosion Model (BSTEM)
during my time at HEC, and this paper covers a major application---probably
the largest RAS-BSTEM model to date---and discusses calibration approaches
and lessons learned. If you're planning on doing bank failure and migration
modeling with HEC-RAS, it's worth a read!
