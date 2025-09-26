---
layout: single
title: "Toronto Bicycle Theft"
date: 2025-08-11
subtitle: 'Analyzing historical bicycle theft in Toronto and predicting correlated geographic features'
excerpt: 'Analyzing historical bicycle theft in Toronto and predicting correlated geographic features'
author_profile: false
tags: [Geospatial Analysis, Machine Learning]
header:
  teaser: /assets/projects/20250910_gerrymandering/heatmapscreenshot.png
---

*tl;dr: Here's where to and not to park your bike in Toronto!  Read on for a full description of the analysis, including identifying key geographic features correlated with high theft*

FULL IFRAME MAP
<figcaption>Heatmap of all police-reported Toronto bike thefts between 2014-2025</figcaption>


# Intro
Just a fun little one!  In 2009, the City of Toronto launched its Open Data Portal - which exposes several key operational datasets from municipal organizations to the public.  In perusing this portal, a dataset that caught my eye was a record of all TPS-reported bicycle thefts in Toronto from 2014 to present.  As an avid cyclist and proud owner of a 1990s marketplace bargain bike, I thought it would be fun to take a look and see if I could identify any insights to help protect my trusty steed.

# Getting the Data
This project started with the cart a bit in front of the horse, as the catalyst was the dataset and I was trying to figure out what we could do with it.  TPS's Bicycle Theft database contains an entry for each of the over 38,000 reports filed since 2014 reporting stolen bicycles.  It includes date, time, and location features for both the incident and report filing, sparsely-populated information on bicycle make, model, and cost (to be fair I couldn't tell you what make my bike is... i just say "yellow"), and status updates for the rare instances when a bike is actually recovered.

I wanted to enhance the dataset a bit, so I passed a subset of recent data through Nominatim - a free open-sourced API for identifying address information based on coordinates.  Nominatim is built on top of OpenStreetMap (OSM - basically a free open-sourced Google Maps database), so I also downloaded a full OSM data dump of Toronto geographic features (parks, restaurants, bars, etc.) to see if I could identify any correlation between theft and the surrounding area.  Put it all together and we're off to the races!

<details>
  <summary>Full code for API nerds</summary>
  WRITE ME
</details>





# The Analysis

