---
title: Likelihood of Finding Null Effects over Time
layout: post
date: 2018-05-20
category: blog
author: amirmasoudabdol
tag: A Better Visualization
headerImage: false
---

A few days ago, we read an interesting paper in our bi-weekly journal club. The paper, “Likelihood of Null Effects of Large NHLBI Clinical Trials has Increased Over Time” [^1], as its title suggests, suggests that more clinical trials are failing to find significant results. Researchers investigated the effect of *pre-registration* on the frequency of finding significant results in large NHLBI studies published from 1974 to 2012. Although very cautionary, they’ve concluded that most studies after 2000 ended up failing to reject the *null* hypothesis. The *Discussion* section — elegantly — listed several other possible factors responsible for this result. Fig 1. summarizes all studies subject to the meta-analysis. The plot clearly shows that after 2000, most studies failed to reject the *null* hypothesis. The visualization is on point and it clarifies the reasoning behind the aforementioned conclusion. 

![Fig 1 of the Kaplan et al. 2015](/assets/posts/a-better-plot-nhlbi-fig-1.png){: class="bigger-image" }
<figcaption class="caption"><b>Fig 1.</b> The history of finding harm or benefit for large NHLBI studies over the year. The vertical line marks the year 2000, where studies had to pre-register their primary outcome in clinical [trials.gov](https://trials.gov) .Figure from Kaplan et al., 2015.</figcaption>

While reading and discussing the paper, we noticed a large gap between studies from 1995 to 2000. The gap may suggest that not many studies have been performed in this period. We also noticed that the plot does not show the *starting year* of the study. So, it is likely that several studies are being started between 1995--2000; however, it's not possible to see this in the plot. *Then, we asked ourselves, if there are studies that are started before 2000, have they been pre-registered?* 

This is an important question as it might affect the conclusion of the paper. If studies that are failing to reject the *null* after 2000 have been started before the mandatory pre-registration, it is possible that the *pre-registration* is not the explaining factor for decrease in the number of *null* cases after 2000. 

So, I’ve tried to re-create the plot. Below, I’ve plotted the same data with a *dumbbell plot*. I drew a line between the *starting year* of a study and its *published year*. Fig 2. suggests that most papers that failed to reject the *null* — after 2000 — are actually started before 1995 where the pre-registration was not necessary.

![Improved Figure](/assets/posts/a-better-plot-nhlbi-fig-2.png){: class="bigger-image"}
<figcaption class="caption"><b>Fig 2.</b> The history of finding primary outcome in NHLBI studies. Each dot represent the publication year of a study, and the line goes back to when the study has been started.</figcaption>

To be clear, I am not at all trying to say that their conclusion is incorrent, I personally find it very interesting that a minor change to the visualization can add new perspective to the problem.

---

[^1]: Kaplan RM, Irvin VL (2015) Likelihood of Null Effects of Large NHLBI Clinical Trials Has Increased over Time. PLOS ONE 10(8): e0132382. [https://doi.org/10.1371/journal.pone.0132382](https://doi.org/10.1371/journal.pone.0132382)