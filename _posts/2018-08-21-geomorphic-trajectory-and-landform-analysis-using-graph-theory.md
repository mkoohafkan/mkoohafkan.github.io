---
layout:     post
title:      Geomorphic trajectory and landform analysis using graph theory
date:       2018-08-21 20:00:00
summary:    I developed a method for automatically generating panel datasets from spatial datasets. It was recently published in Progress in Physical Geography. Check it out!
categories: horntooter dataphile graph-theory geomorphology remote-sensing manuscript
commentIssueId: 34
color: darken-red
---

A big chunk of my research has focused on studying emergent
sandbars in large sand-bedded rivers like the Missouri River.
I first got involved with the work through a short-term contract
with the USACE 
[Hydrologic Engineering Center](http://www.hec.usace.army.mil/). 
Stanford Gibson was collaborating with the USACE Omaha District and the 
[Missouri River Recovery Program](http://moriverrecovery.usace.army.mil) 
to provide some additional sediment transport 
and geomorphology expertise. Part of his work involved generating 
some maps and statistics of sandbar change based on classified
satellite imagery (classified as in 
[Object-Based Image Analysis](https://pubs.er.usgs.gov/publication/70038548), 
not as in top-secret).
I was already looking for an excuse to get involved with HEC, and my
background in GIS made me a pretty good fit for the task.

I completed the project without much trouble, building Stan a GIS toolbox
for processing the imagery and generating statistics on sandbar area. The 
work was tricky because the imagery captured sandbars at a wide range of
river conditions. Some images captured the river at low-flow conditions,
showing a river with huge sandbars and large swathes of bare sand; others
captured the river at high-flow conditions, when most of the sandbars were 
drowned and only the vegetated high ground of the largest bars remained
above water. To make matters worse, each image captured only a fraction of 
the study area; getting a complete snapshot of the system required a lot of
effort chopping and splicing images at similar flows to get a complete picture. 
I was proud of the work I did, but I had this nagging feeling
that there had to be a better way.

That better way turned into a chapter of my dissertation and my latest manuscript,
published to [Progress in Physical Geography](https://doi.org/10.1177/0309133318783143)
(the accepted version is also available [here](/docs/2018-koohafkan-gibson-ppg-accepted.pdf)).
In the paper, I describe a novel method for generating time series of landforms
from sets of fully- or partially-overlapping snapshots of a system. The method 
automatically links observations of individual landforms across images---even 
in cases where landforms fragment, merge, migrate, or become temporarily 
obstructed from view---and generates time series for individual landforms. The
method is powered by graph theory, which I use to create a data structure 
that represents connections between features across snapshots.

This is a methods paper, so I focus on how the procedure compares to current 
practices of analyzing spatial data.
I do some fairly basic demonstrations of analyzing the panel datasets the method 
generates, and explore the potential for applying some of the more advanced aspects of
graph theory for an alternative approach to analyzing spatial data. But I'm not done
yet! I've been applying the method to an analysis of the imagery that started 
this whole thing, and---at the risk of sounding immodest---the results are powerful. 
On top of that, I'm also pursuing an entirely new analysis that uses the datasets 
I developed for the 
[NOAA Habitat Blueprint for the Russian River]({% post_url 2016-04-01-a-shiny-approach-to-data-collaboration %}) 
to demonstrate the method's utility in ecological studies. In this new work, I also extend
the method to capture not just connections between features between snapshots, but also how 
relationships between features in the same layer (e.g., feature adjacency) change through 
time. 

I'm looking forward to sharing my analysis of the emergent sandbars of the Missouri 
River later this year. I think the method is a powerful new tool that can be applied
to almost any field interested in tracking landscape features through time---whether
they are actual landforms, habitat patches, or even more abstract spatial structures.
If I haven't convinced you to feel the same way, try reading
the [paper](/docs/2018-koohafkan-gibson-ppg-accepted.pdf)!
