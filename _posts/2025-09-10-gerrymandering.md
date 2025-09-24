---
layout: single
title: "Gerrymandering!"
post_header: true
subtitle: 'Creating an automated method to achieve the "impossible" 11-2 NC split'
excerpt: 'An automated method to achieve the "impossible" 11-2 NC split'
author_profile: false
tags: [Geospatial Analysis]
header:
  teaser: /assets/projects/20250910_gerrymandering/heatmapscreenshot.png
date: 2025-09-10
---

*tl;dr: I tried my hand at "packing and cracking" state districts in an attempt to out-gerrymander north Carolina Republicans for the 2016 House election.  Check out my final district map below, and read on for the full method to develop this process.*

<iframe src="{{ site.baseurl }}/assets/projects/20250910_gerrymandering/finalmap.html" height="400" width="100%"></iframe>
<figcaption>My final district map to achieve an 11-2 Republican majority in the 2016 North Carolina House election</figcaption>



## Background
Leading up to the 2016 U.S. elections, North Carolina's popular vote was relatively 50/50 between Democrats and Republicans, and historically the state's House representation had been evenly split between the two parties.  In the lead up to the 2016 election, Republican lawmakers managed to pass a gerrymandered redistricting map that led Republicans to secure a 10-3 victory in the 2016 House election.  In 2018, the 10-3 district map was struck down by U.S. Federal courts, who deemed it an unconstitutional partisan gerrymander.  The case has widely been criticized as an aggregious example of partisan gerrymandering, and an example of gerrymandering's negative impact on fair democratic process.

In the lead up to the election, Republican Rep. David Lewis was famously cited as saying "I propose that we draw the maps to give a partisan [10-3 split], because I **do not believe it's possible to draw a map with [an 11-2 split]**."  ...I don't know about you, but that sounds like a challenge to me!

<br>
<figcaption>*I feel obliged here to give a disclaimer that in the real world, I am not remotely in favour of partisan gerrymandering, and my intention with this project is to gamify and illustrate the absurdity of the concept.  Here in Canada, our election maps are drawn by joint panels of statisticians and retired judges - which, when you hear it, wow that makes so much more sense, eh?</figcaption>





## Getting the Data
First thing needed to start this was the 2016 election results.  I recognize this is obviously much more definitive voter demographic information than would have been available to Republican lawmakers at the time - so sue me.  The MIT Election Data and Sciences Lab (MEDSL) came in handy here, with a full dataset of North Carolina results by precinct for 2016 elections.  The second piece was a map that I could put these results on.  The University of Florida's Voting and Election Science Team (VEST) handled this one, with a 2016 archive of SHP file geometries for North Carolina election precinct boundaries.

Compiling the dataset involved mapping the MEDSL election results onto the VEST precinct geometries.  I had to make a couple assumptions to do this (outlined below), but then we were off to the races!  Below is the official "10-3" district mapping that was used in the 2016 election, as well as a heatmap and results from each district:

![]({{ site.baseurl }}/assets/projects/20250910_gerrymandering/basemap.png)
<figcaption>Official district mapping for the 2016 North Carolina House election</figcaption>

![]({{ site.baseurl }}/assets/projects/20250910_gerrymandering/basemapresults.png)
<figcaption>2016 election results from the official North Carolina district mapping</figcaption>

XXXX
<figcaption>Heatmap of compiled 2016 election data showing geographic voter composition</figcaption>


<details>
  <summary>Full code for nerds</summary>
  Importing and cleansing csvs, and joining results together.  In this case, both datasets featured a precinct + jurisdiction granularity:

  {% highlight python %}
results = pd.read_csv("nc_medsl_16_general/nc_medsl_16_general.csv")  #2016 elections results from MEDSL (2016)
results = results[(results['office']=="US House") & (results['stage']=='gen')]  #dataset has lots of elections, this is the one we care about
results['district'] = results['district'].astype(float).astype(int)  #weirdly a mix of strings and floats here...

precincts=gpd.read_file("nc_2016/nc_2016.shp")  #precinct SHP from VEST (2016)
precincts.head(10)  #hmm.... doesn't seem to have house election results
precincts = precincts[['PREC_ID', 'ENR_DESC', 'COUNTY_NAM', 'COUNTY_ID', 'geometry']]  #this is really all thats good from here
precincts = precincts.dropna(subset=['geometry'])

#compile a clean combined dataset
results_pvt = results.pivot_table(index=['precinct', 'jurisdiction'],  #for this analysis, precinct/juris is the primary key between these tables.  districts CAN (and do) cross precinct lines, so can't easily be mapped.  the purpose of THIS df is to let you form new ones
                                  columns='party', 
                                  values='votes', 
                                  aggfunc='sum', 
                                  fill_value=0)
unique_parties = list(results_pvt.columns)  #hold this list for later to dynamically identify all the party columns we added
results_pvt['total_votes'] = results_pvt.sum(axis=1) #add a quick total column
results_pvt = results_pvt[results_pvt['total_votes']>0]  #exclude lines with no votes

results_pvt = results_pvt.reset_index()  #pull the index columns back as real columns
results_pvt = results_pvt.merge(precincts, how='left', left_on=['precinct', 'jurisdiction'], right_on=['PREC_ID', 'COUNTY_NAM'])  #merging on name feels fragile for my liking... I just don't see any other matches between these datasets

#it's imperfect, but for visual purposes let's add the most-used district in each polygon (this won't matter for new district creation)
_grp = results.groupby(['precinct', 'jurisdiction', 'district'], as_index=False)['votes'].sum()
_topdist = _grp.groupby(['precinct', 'jurisdiction'], as_index=False, ).agg(idxmax=('votes','idxmax'))
_topdist['top_district'] = _topdist['idxmax'].map(_grp['district'])

results_pvt = results_pvt.merge(_topdist.drop(columns=['idxmax']), how='left', on=['precinct', 'jurisdiction'])
results_pvt.head(5)
  {% endhighlight %}


  Unfortunately, not all of the MEDSL election results match the physical <em>geographic</em> precincts we see from VEST. Many of the votes submitted in this election were mail-in or advance polling, which are not attributed to geographical precincts -  only the district in which they were cast.  I had to make the assumption that the geographic dispersion of advance pollers within a district would be similar to in-person voters of the same district - hence, allocate the advance polling votes within each district to physical geographies proprtionally based on the geographic distribution of in-person voters in that district.  Perfect? No. Inevitable? Seemed so:

  {% highlight python %}
no_geo = results_pvt[results_pvt['geometry'].isna()]
w_geo = results_pvt[~results_pvt['geometry'].isna()]

#create a mapping of the distribution of IN-PERSON jurisdiction votes into the geometric precincts
juris_prcmap = w_geo.pivot_table(index='jurisdiction', columns='precinct', values='total_votes', aggfunc='sum', fill_value=0)
juris_totals = juris_prcmap.sum(axis=1)
juris_prcmap = juris_prcmap.div(juris_totals, axis=0)

nogeo_juris = no_geo.groupby('jurisdiction')[unique_parties].sum()

#allocate ADVANCE POLLING votes proportionally based on the above
mappeds = {}
for juris, votes in nogeo_juris.iterrows():
    _prcmap = juris_prcmap.loc[juris]
    _df = pd.DataFrame(
        np.array(_prcmap).reshape(-1,1) * np.array(votes).reshape(1,-1),  #spin up a 2d array of votes per party per allocated prc
        index = _prcmap.index,
        columns = ['nogeo_'+i for i in votes.index]
    )
    mappeds[juris] = _df

nogeo_mapped = pd.concat(mappeds, names = ['jurisdiction', 'precinct'])
nogeo_mapped = nogeo_mapped[nogeo_mapped.sum(axis=1)>0]  #drop unused prc mappings
nogeo_mapped = nogeo_mapped.reset_index()

#combine the two datasets
mapped_results = w_geo.merge(nogeo_mapped, how='outer', on=['jurisdiction', 'precinct'])
mapped_results = mapped_results.fillna(0)

#combine geo and non/geo into one column (and a new total)
for p in unique_parties:
    mapped_results['total_'+p] = mapped_results[p] + mapped_results['nogeo_'+p]

mapped_results['combined_total_votes'] = mapped_results[['total_'+p for p in unique_parties]].sum(axis=1)
mapped_results['pct_mapped'] = mapped_results['total_votes'] / mapped_results['combined_total_votes']

for p in unique_parties: mapped_results['pct_'+p] = mapped_results['total_'+p] / mapped_results['combined_total_votes']

mapped_results = mapped_results.drop(columns = unique_parties + ['nogeo_'+p for p in unique_parties] + ['total_votes', 'PREC_ID', 'COUNTY_NAM'])
mapped_results = mapped_results.sort_values(['precinct', 'jurisdiction'])
mapped_results = gpd.GeoDataFrame(mapped_results, geometry='geometry')  #convert to gpd
mapped_results = mapped_results.to_crs("WGS84")  #convert to lat/lon crs
mapped_results['precinct_juris'] = mapped_results['precinct'] + mapped_results['jurisdiction']  #create a single primary key column
mapped_results.head(5)
  
  {% endhighlight %}
</details>






## The Analysis

### Intro
Okay - so how to start approaching a problem like this?  This simple-sounding project ended up spiralling into many iterations of failed attempts and taxing overnight runs for my laptop fan.  I won't fully go into every wacky idea I tried along the way, but I'll outline some of the key stages and learnings.  My key requirements for a solution here were that a final district map had to:

- Allocate the existing area of North Carolina into 13 ~equally sized districts
- Maintain geographical contiguity across each of the 13 districts
- Produce "reasonable looking" geometries 
- Achieve a "safe" majority election win for Republicans in 11 of the 13 districts

<details>
  <summary>Full definitions (for nerds)</summary>
  For "equally sized", I deferred to the official districts for an acceptable range of variation - which was 323k-409k votes per district.<br><br>  
  Similarly, I deferred to the official election results for what lawmakers at the time considered a "safe" majority - in this case the lowest-performing Republican district won with a 52.3% margin (I did end up aiming a little higher given my benefit of hindsight). 
</details>



### Attempt #1: Full Monte Carlo
My first theory for solving this was that: if I just randomly modified the starting district map while maintaining size and contiguity requirements, I would eventually discover every possible legal configuration and could then just pick my favourite.  I set up a script to run through this overnight and got the following:

![]({{ site.baseurl }}/assets/projects/20250910_gerrymandering/20250828_MonteCarlo_gif_5xspeed-ezgif.com-crop.gif)
<figcaption>Attempt #1 at using a Monte Carlo method to randomly iterate through possible configurations... not super successful</figcaption>




<details>
  <summary>Full code for nerds</summary>
  Conceptually, the idea was to pick a random district, identify a precinct within that district elligible to move elsewhere, and swap it to a neighbouring district.  My assumption (spoiler - <em>incorrect</em> assumption) was that any precinct that sat on the boundary between two districts could safely be moved to either without breaking contiguity.  I then ran a check to confirm that moving the precinct did not exceed our allowable district sizes on either district, and made the swap:

  {% highlight python %}
  #the idea will be to start with the current map and just randomly give a border precinct to a neighbouring precinct (where able), and see what that gives us
#this can go through ~100/minute, of which 70% are valid swaps (160)
district_assignments = []  #to house an ordered list of district assignments
district_results = [] #to house a scoring metric - we will choose % dem
test_total_cts = []


with warnings.catch_warnings():
    warnings.simplefilter("ignore")  #mildly invalid geometries in buffer products, run without warnings

    for _ in range(1000000):
        if _ >0:  #to write the initial state
            _fromdist = random.randint(1,13)  #pick a random district
    
            _fromprecs = _moving_results[_moving_results['district']==_fromdist]  #get the subset of prec-juris in this district
            _fromborder = _fromprecs.geometry.union_all().boundary  #identifythe outer boundary to find permimeter pjs
            _fromcandidates = _fromprecs[_fromprecs.geometry.intersects(_fromborder)]  #subset of perimeter pjs
            _mover = _fromcandidates.iloc[[random.randint(0,len(_fromcandidates)-1)]]  #pick a random one  #supposedly the double brackets keep it a gdf
            
            _ddistmask = _moving_results['district']!=_fromdist
            _touchmask = _moving_results.geometry.intersects(_mover.geometry.iloc[0].buffer(200))
            _tocandidates = _moving_results[_ddistmask & _touchmask]
            
            if len(_tocandidates) == 0: continue  #abort this loop
                
            _todist = _tocandidates.iloc[random.randint(0,len(_tocandidates)-1)]['district']
            _newfromsize = _moving_results[_moving_results['district']==_fromdist]['combined_total_votes'].sum() - _mover.iloc[0]['combined_total_votes']
            _newtosize = _moving_results[_moving_results['district']==_todist]['combined_total_votes'].sum() + _mover.iloc[0]['combined_total_votes']
            
            if _newfromsize < dist_minsize or _newtosize > dist_maxsize: continue  #not allowed move
            
            _moving_results.loc[_mover.index, 'district'] = _todist  #move it
        
        #write results outside the if
        district_assignments.append(np.array(_moving_results['district']))  #back up this current state, np.array for size
        
        _newresults = _moving_results.groupby('district')[['total_democratic', 'combined_total_votes']].apply(
            lambda x: pd.Series({"pct_dem": x['total_democratic'].sum() / x['combined_total_votes'].sum()})
        )  #only storing 1D array, important  #default sorting is by index ascending (just remember index is 0-12 not 1-13)
        district_results.append(np.array(_newresults['pct_dem']))
    
        if _%1000==0:
            print(f"{_} complete, {len(district_assignments)} successful")
            np.savez_compressed("montecarloresults_npzip", district_assignments=district_assignments, district_results=district_results)
    
print(f"all complete, {len(district_assignments)} successful")
np.savez_compressed("montecarloresults_npzip", district_assignments=district_assignments, district_results=district_results)  #final save
  {% endhighlight %}

  Clearly, this did not work for multiple reasons.  Specifically for contiguity though, the issue was the assumption that a bordering precinct could always be moved without breaking contiguity.  If the precincts were a grid of perfect squares (how I drew them when I came up with this lol) it would be fine, but with complex real geometries, a precinct can touch the border of a district while still being integral for continuity.  Once contiguity is broken once, the effect cascades as now the "island" starts trying to grab precincts bordering it.
</details>





As you can probably see, this didn't work.  The first obvious problem was that my rule for maintaining contiguity was a little too soft (and quickly spiraled out of control), but actually the bigger problem was runtime.  After running for nine hours, the district map had not changed that significantly.  Some quick back of the envelope math revealed that, with 2,700 precincts, the number of valid maps was likely in the magnitude of **1e100+**, and trying to find them all through brute force was not going to be realistic.




### Attempt #2: Targeted Construction
Given the massive number of potential solutions and the heavy computation required to manipulate these contiguous geometries, I realized I was going to need to be a lot more targeted in my approach.  My second idea was that I would simply nail it the first time, and build a map from scratch.  

The core idea of gerrymandering is "packing and cracking" - i.e. consolidating all of your opponents votes into a small number of districts where they win with an excessive majority ("packing"), and then diluting the rest of their votes across the remaining districts such that they *just marginally* lose in each.  

Conceptually, my strategy was to first pack then crack - presuming that as long as I could sufficiently pack two districts, the remaining 11 would be easy (*spoiler - they were not*).  I let the high-level demographics inform the targets for each stage.  Overall, Republicans held **53%** of the popular vote.  Therefore, in order to secure a 57% victory in 11 of the 13 districts, the remaining two districts would need to be about **70% Democratic**.

##### Packing!
So - how to find legal districts that are comprised of 70% Democratic voters?  My approach was to, like a sculptor, start with the entire "block" of the state and intelligently whittle it down until only a highly condensed district remains.  If you're a nerd like me who cares, the full specifics of this logic and development process are below, but after several iterations I was able to achieve a successful 70% result:

![]({{ site.baseurl }}/assets/projects/20250910_gerrymandering/loserdistrict1-ezgif.com-speed.gif)
<figcaption>The first successful "pack"!  Creation of a 70% Democratic district</figcaption>

<details>
  <summary>Full code and method for nerds like me</summary>
  WRITE ME
</details>







Now, all that was left was to do it again!  In reality, this process involved a lot of back and forth between the two districts to ensure that the first wasn't "too greedy" and sufficient valid options remained for the second.  But, after a lot of failed attempts and overnight runs we had two valid districts averaging **69% Democratic**:

![]({{ site.baseurl }}/assets/projects/20250910_gerrymandering/loserdistrict2-ezgif.com-speed.gif)
<figcaption>Construction of the second Democratic district</figcaption>

<details>
  <summary>Full method</summary>
  <img src = "{{ site.baseurl }}/assets/projects/20250910_gerrymandering/doitagain.jpeg" width="50%">
</details>






##### Cracking!
Okay - so now the easy part right?  Having constructed two 69% Democratic districts, the remaining state averages **57% Republican**, so it should just be a simple exercise of divvying it up.  It was not.  Perhaps unsurprisingly, the local concentrations of Republican votes and the constraints of maintaining contiguity as the map filled in made this extremely difficult (specifically for the later districts).

Again, if you're nerdy like me there's a full outline of the method below, but the high-level approach I employed here is probably the same one a five year-old would - to just start at the left and carefully add one adjoining precinct at a time.  Obviously, there were some iterations on this approach and a couple clever tricks for deciding which precinct to add next.  After probably way too many attempts and iterations, I was able to get pretty close - although the Republican margin on the 13th district wasn't quite as "safe" as I wanted it to be:

![]({ site.baseurl }}/assets/projects/20250910_gerrymandering/winnerdistricts-ezgif.com-speed.gif)
<figcaption>Construction of the remaining 11 Republican districts</figcaption>

![]({{ site.baseurl }}/assets/projects/20250910_gerrymandering/beforeswapresults.png)
<figcaption>2016 NC House election results using the 13 districts above</figcaption>

<details>
  <summary>Full code and method for nerds like me</summary>
  WRITE ME
</details>






### Attempt 2.5: Monte Carlo Again??
After probably far too long of beating my head against the wall trying to get the first pass above to be perfect, I decided to revisit our old friend Monte Carlo.  I realized that the target construction above was very close, and that the easiest thing might just be to employ a more targeted swapping approach to "clean up" the districts and get it across the line.  The full method is below, but the high-level approach was to idenitfy swaps that were *mutually beneficial* to the overall map, with the objective of getting all Republican districts to at least a 55% majority:

![]({{ site.baseurl }}/assets/projects/20250910_gerrymandering/swapcleanup-ezgif.com-speed.gif)
<figcaption>A final "swapping" cleanup on the district map to balance Republican margins</figcaption>

<details>
  <summary>Full code for nerds</summary>
  WRITE ME
</details>





And with that, we have officially achieved the "impossible" 11-2 Republican split!  Below is the final district mapping and district election results for the 2016 NC House election:

![]({{ site.baseurl }}/assets/projects/20250910_gerrymandering/finalmap.png)
<figcaption>The final optimized district mapping to achieve an 11-2 Republican split in the 2016 NC House election</figcaption>

![]({{ site.baseurl }}/assets/projects/20250910_gerrymandering/finalmapresults.png)
<figcaption>2016 House election results by NC district using the optimized mapping - Republicans win in 11 of 13 districts with a margin or 55% or higher in each</figcaption>




## Final Thoughts















