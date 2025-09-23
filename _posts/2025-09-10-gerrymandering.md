---
layout: single
title: "Gerrymandering!"
post_header: true
subtitle: 'Creating an automated method to achieve the "impossible" 11-2 NC split'
excerpt: 'An automated method to achieve the "impossible" 11-2 NC split'
author_profile: false
tags: [Geospatial Analysis]
header:
  teaser: /assets/projects/20250910_gerrymandering/finalmap.png
date: 2025-09-10
---

*tl;dr: I tried my hand at "packing and cracking" state districts in an attempt to out-gerrymander north Carolina Republicans for the 2016 House election.  Check out my final district map below, and read on for the full method to develop this process.*

<iframe src="{{ site.baseurl }}/assets/projects/20250910_gerrymandering/finalmap.html", height=400, width=100%></iframe>
<figcaption>My final district map to achieve an 11-2 NC split in the 2016 House election</figcaption>



## Background
Leading up to the 2016 U.S. elections, North Carolina's popular vote was relatively 50/50 between Democrats and Republicans, and historically the state's House representation had been evenly split between the two parties.  In the lead up to the 2016 election, Republican lawmakers managed to pass a gerrymandered redistricting map that led Republicans to secure a 10-3 victory in the 2016 House election.  In 2018, the 10-3 district map was struck down by U.S. Federal courts, who deemed it an unconstitutional partisan gerrymander.  The case has widely been criticized as an aggregious example of partisan gerrymandering, and an example of gerrymandering's negative impact on fair democratic process.

In the lead up to the election, Republican Rep. David Lewis was famously cited as saying "I propose that we draw the maps to give a partisan [10-3 split], because I do not believe it's possible to draw a map with [an 11-2 split]."  ...I don't know about you, but that sounds like a challenge to me!

<figcaption>I feel obliged here to give a disclaimer that in the real world, I am not remotely in favour of partisan gerrymandering, and my intention with this project is to gamify and illustrate the absurdity of the concept.  Here in Canada, our election maps are drawn by joint panels of statisticians and retired judges - which, when you hear it wow that makes so much more sense eh?</figcaption>



## The Analysis

