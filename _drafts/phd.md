---
title: "Ph.D. Project"
layout: post
category: project
projects: true
hidden: true
author: amirmasoudabdol
date: 2018-05-08 00:00
headerImage: false
---

<!-- ![](/assets/projects/PhD_Cover.png){: class="bigger-image" } -->

### Toward Reverse Engineering Spatiotemporal Gene Regulartory Networks of the *Nematostella vectensis*

Over the last decade, the sea anemone *Nematostella vectensis* has become a popular model to study bilaterian evolution, development and more recently also regeneration. Understanding genetic interactions during the early development of *N. vectensis* is the first step toward unveiling the details of its early developmental processes, e.g., polarization, the formation of the *blastula* and initiation of the *gastrulation* process. Furthermore, the collective knowledge of gene interactions allows researchers to speculate about possible *Gene Regulatory Networks* (GRNs) governing each process. The knowledge of gene interactions also provides an opportunity to reverse engineer gene interactions in a GRN model which potentially leads to detailed understanding of the early developmental processes, e.g., pattern formation and the mechanics of gastrulation. There is still a limited amount of knowledge available regarding *N. vectensis* gene interactions and possible GRNs involved in each developmental process. This thesis introduces a method for extracting spatial gene expression profiles from *in situ* hybridization images of *N. vectensis* embryo. My collaborators and I have introduced a systematic procedure to combine and process the available data from different sources (e.g., *in situ* and *qPCR*) in order to understand gene interactions and reconstruct testable hypotheses for GRNs controlling development.

In the case of *N. vectensis*, spatial gene expression profiles are available in the form of *in situ* hybridization images of the embryo. The changing morphology of *N. vectensis* embryo during development imposes a crucial problem for detecting the expression of the genes from the *in situ* images. Therefore, we developed an algorithm to detect and track morphological properties of the embryo from the *in situ* images and extract the gene expression profiles from the images of the embryo during *blastula* and *gastrula* stages.

Thereafter, we used the extracted spatial gene expressions profiles from *in situ* images to study *N. vectensis* gene interactions. By clustering the spatial data, we showed that we could detect functional regions of the embryo during the *blastula* and *gastrula* stages. Similarly, we discovered significant developmental events by clustering the temporal genes expressions, in the form of qPCR time series. Furthermore, we introduced a method for merging the clustering results from spatial and temporal datasets by which we can group genes that are expressed in the same region and at the same time in the embryo. We demonstrated that the merged clusters could be used to identify gene interactions involved in various processes and also to predict possible activators or repressors of any gene in the dataset.  Finally, we validated our methods and results by predicting the repressor effect of *NvErg* on *NvBra* in the central domain during the gastrulation that has recently been confirmed by functional analysis.

Being able to provide a list of gene interactions, we could propose multiple models of possible GRNs involved in different developmental processes. However, the computationally intensive optimization procedure of reverse engineering GRNs has challenged us to improve the optimization process before tackling the unknown world of *N. vectensis* GRNs. Therefore, we developed and tested a new hybrid optimization approach by combining two optimization algorithms, the *Scatter Search* and the *Simulated Annealing*. With the new hybrid method, we provided a powerful exploratory method for reverse engineering GRNs in organisms with a changing morphology during development.