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

*tl;dr: I tried my hand at "packing and cracking" state districts in an attempt to out-gerrymander North Carolina Republicans for the 2016 House election.  Check out my final district map below, and read on for the full method to develop this process.*

![]({{ site.baseurl }}/assets/projects/20250910_gerrymandering/finalmap.png)
<figcaption>My final district map to achieve an 11-2 Republican majority in the 2016 North Carolina House election</figcaption>



## Background
Leading up to the 2016 U.S. elections, North Carolina's popular vote was relatively 50/50 between Democrats and Republicans, and historically the state's House representation had been evenly split between the two parties.  In the lead up to the 2016 election, Republican lawmakers managed to pass a gerrymandered redistricting map that led Republicans to secure a 10-3 victory in the 2016 House election.  In 2018, the 10-3 district map was struck down by U.S. Federal courts, who deemed it an unconstitutional partisan gerrymander.  The case has widely been criticized as an aggregious example of partisan gerrymandering, and an example of gerrymandering's negative impact on fair democratic process.

In the lead up to the election, Republican Rep. David Lewis was famously cited as saying "I propose that we draw the maps to give a partisan [10-3 split], because I **do not believe it's possible to draw a map with [an 11-2 split]**."

I don't know about you, but that sounds like a challenge to me!

<br>
<figcaption>*I feel obliged here to give a disclaimer that in the real world, I am not remotely in favour of partisan gerrymandering, and my intention with this project is to gamify and illustrate the absurdity of the concept.  Here in Canada, our election maps are drawn by joint panels of statisticians and retired judges - which, when you hear it, wow that makes so much more sense, eh?</figcaption>





## Getting the Data
First thing needed to start this was the 2016 election results.  I recognize this is obviously much more definitive voter demographic information than would have been available to Republican lawmakers at the time - so sue me.  The MIT Election Data and Sciences Lab (MEDSL) came in handy here, with a full dataset of North Carolina results by precinct for 2016 elections.  The second piece was a map that I could put these results on.  The University of Florida's Voting and Election Science Team (VEST) handled this one, with a 2016 archive of SHP file geometries for North Carolina election precinct boundaries.

Compiling the dataset involved mapping the MEDSL election results onto the VEST precinct geometries.  I had to make a couple assumptions to do this (outlined below), but then we were off to the races!  Below is the official "10-3" district mapping that was used in the 2016 election, as well as a heatmap and results from each district:

![]({{ site.baseurl }}/assets/projects/20250910_gerrymandering/basemap.png)
<figcaption>Official district mapping used for the 2016 North Carolina House election</figcaption>

![]({{ site.baseurl }}/assets/projects/20250910_gerrymandering/basemapresults.png)
<figcaption>2016 election results from the official North Carolina district mapping</figcaption>

<iframe src="{{ site.baseurl }}/assets/projects/20250910_gerrymandering/baseheatmap.html" height="400" width="100%"></iframe>
<figcaption>Interactive heatmap of compiled 2016 election data showing geographic voter composition</figcaption>




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
Given the massive number of potential solutions and the heavy computation required to manipulate these geometries, I realized I was going to need to be a lot more targeted in my approach.  My second idea was that I would simply nail it the first time, and build a map from scratch.  

The core idea of gerrymandering is "packing and cracking" - i.e. consolidating all of your opponents votes into a small number of districts where they win with an excessive majority ("packing"), and then diluting the rest of their votes across the remaining districts such that they *just marginally* lose in each.  Conceptually, my strategy was to first pack then crack.  My presumption was that as long as I could sufficiently pack two districts, the remaining 11 would be easy (*spoiler - they were not*).  I let the high-level results inform the targets for each stage.  Overall, Republicans held **53%** of the popular vote.  Therefore, in order to secure a 57% victory in 11 districts, the remaining two districts would need to be about **70% Democratic**.

##### Packing!
So - how to find legal districts that are comprised of 70% Democratic voters?  My approach was to, like a sculptor, start with the entire "block" of the state and slowly whittle it down until only a highly condensed district remains.  If you're a nerd like me, the full specifics of this logic and development process are below, but after several iterations I was able to achieve a successful 70% result:

![]({{ site.baseurl }}/assets/projects/20250910_gerrymandering/loserdistrict1-ezgif.com-speed.gif)
<figcaption>The first successful "pack"!  Creation of a 70% Democratic district</figcaption>




<details>
  <summary>Full code and method for nerds like me</summary>
  The script that generated this is one big nasty loop, so I'll break down the core pieces then show the full code block.<br><br>

  Fundamentally, the idea was to start with a GeoDataFrame of all available precincts and to iteratively drop precincts until a legal district remained. At each iteration, I would identify the precincts that bordered the outer perimeter of the remaining geometry as candidates to be dropped:

  {% highlight python %}
target = 0.30

_available = optimized_districts[optimized_districts['district']==0].copy()  #initially, everything is an option for d1
_currgeom = _available.union_all()  #this will permanently house the unioned geometry (to avoid recalc)

print(f"[{datetime.datetime.now():%I:%M:%S %p}] start")

_c = 1  #count loops   
while not (_available['combined_total_votes'].sum() <= dmax and pcttarget(_available) <= target):
    _border = _currgeom.boundary
    _touchmask = _available.geometry.intersects(_border)
  
    _edgeprecs = _available[_touchmask]  #available edge pieces

    if len(_edgeprecs)==0:  #if there's nothing left to legally drop
        print(f"no solution found ({_c-1} iterations)")
        break
  {% endhighlight %}



  Next, I calculated a "score" for each precinct describing how beneficial dropping it would be to the remaining district.  My goal was to create a scoring metric that would balance getting the district to the target demographic, while also maintaining a "reasonable looking" geometry.  For "reasonable looking", I used a metric I dubbed "branchyness", which is the ratio of the geometry's permimeter (squared to maintain units) over area.  More perimeter per area = weirder shape.<br><br>
  
  For getting to target demographic, I just calculated what the percent Republican makeup of the remaining district would be without this precinct.  To have this balance with the "branchyness" metric, I <strong>squared</strong> the difference from target.  This meant that when the district was close to demographic target, the metric would be near-zero and the overall scoring would prioritize creating an intuitive geometry - but when the district was far from demogrphic target, the squared difference would balloon and dominate.<br><br>

  I also included a bit of a "look-ahead" score, which was the same as above but considered everything within several kilometers of each precinct.  This helped the algorithm avoid local maximums and prioritize <em>general areas</em> that were advantageous.  I set this look-ahead distance to gradually shrink as the district closed in on its final area.<br><br>

  Putting it all together:

  {% highlight python %}
_sortscores = [] #[idx, score, geomin]
for i in _edgeprecs.index:
    _igeom = _available.loc[i].geometry
    _geomin = _currgeom.difference(_igeom)
    _branchyness = _geomin.length**2 / _geomin.area

    _t = 30e3 * (_available['combined_total_votes'].sum()-323e3) / 4e6  #lookahead distance shrinks as it gets smaller (starts ~4.5M votes)
    minx, miny, maxx, maxy = _igeom.bounds
    _fastbuffer = (minx-_t, miny-_t, maxx+_t, maxy+_t)  #apparently much faster than .buffer(10e3)

    _touchi = _available.iloc[_available.sindex.intersection(_fastbuffer)]  #do this touchi thing again
    _ipct, _touchipct = pcttarget(_available.drop(i)), pcttarget(_available.drop(_touchi.index))
    _ipctscore, _touchipctscore = abs(_ipct - target)**2*1e4, abs(_touchipct - target)**2*1e4
    
    _sortscore = _branchyness + _ipctscore + 0.5*_touchipctscore
    _sortscores.append([i, _sortscore, _igeom])

_sortscores = sorted(_sortscores, key=lambda x: x[1])  #lowest score at the top - these are ones where dropping produces a better (lower score) result
  {% endhighlight %}




  Next was legality checks.  Basically, the idea was to take the most favourably-scored precinct and validate that dropping it would not break contiguity or exceed our allowable district size.  If either was violated, the precinct would be flagged as "illegal" so it wouldn't be prioritized next time - otherwise, the precinct would be dropped from the district.<br><br>

  I added a clause that if a desirable precinct to drop divided the remaining district (broke contiguity), before calling the drop illegal it would first consider the possibility of dropping the <em>entire</em> fragmented branch.  You can see this happen a few times in the gif above.  I also added a clause to maintain contiguity of the <em>remaining</em> state geometry (to avoid sewering the later steps), however intentionally <strong>disabled</strong> this rule until the final iterations (<1M votes remain) to allow the algorithm to quickly drop from both ends at the beginning.  Again, in the gif above you can see where this rule kicks in and a previously dropped area is added <em>back in</em> to create contiguity of the remaining state.<br><br>
  
  To save runtime on the computationally-heavy previous scoring step above, I allowed the algorithm to drop more than just one precinct while it was at this step (in this case, up to the best-scored 50% of border precincts):

  {% highlight python %}
_ndrop = max(int(len(_edgeprecs)*0.33), 1) #try some speed, drop x% of border or 1
        
for l in _sortscores[:_ndrop]:    
    i = l[0]
    if l[0] not in _available.index: continue  #in case idx dropped in branch, skip all 
    
    _igeom = l[2]
    _geomin = _currgeom.difference(_igeom)  #**need to recalculate this!  with multiple dropped since scoring, it's out of date
    if isinstance(_geomin, MultiPolygon): _validgeoms = [g for g in _geomin.geoms if g.area>23e3]  #some individual precs genuinely have slivers

    if _unifyremaining: #only calculate this when needed
        _geomout = _remaingeom.union(_igeom)  
        if isinstance(_geomout, MultiPolygon): _validout = [g for g in _geomout.geoms if g.area>23e3]  #same deal
    
    #----------legality checks:---------------------------
    if _unifyremaining and isinstance(_geomout, MultiPolygon) and len(_validout)>1: markillegal(i, "fractures remaining")  #this one only gets checked at the end
    
    elif isinstance(_geomin, MultiPolygon) and len(_validgeoms)>1:  #if dropping this produces a multipolygon (ie fractures the contiguity)... don't immediately say the drop is illegal...
        if len(_validgeoms)>2: markillegal(i, "3+ polygons")  #don't bother with the complexity of 3+ polygons
        else:
            _poly1 = _available[_available.geometry.intersects(_validgeoms[0])]  #identify the two district polygons
            _poly2 = _available[_available.geometry.intersects(_validgeoms[1])]

            if _poly1['combined_total_votes'].sum() > _poly2['combined_total_votes'].sum(): _bigger, _smaller = _poly1, _poly2
            else: _bigger, _smaller = _poly2, _poly1  #name them "bigger" and "smaller" for ease

            _pctbigger = pcttarget(_bigger)  #pct dem in each for ease
            _pctsmaller = pcttarget(_smaller)

            #run some checks to decide if i is illegal to remove, or we should gas the whole _smaller branch with it
            if _bigger['combined_total_votes'].sum() < dmin: markillegal(i, "too small without branch")  #if removing this compromises dmin... can't do it
            elif _pctsmaller < _pctbigger: markillegal(i, "branch closer to target")  #if the branch is higher % than the rest, leave it (coudl change later)
            else: 
                _todrop = [i] + list(_smaller.index)
                for p in _todrop: dropprec(p, f"dropped with branch {i}")  #if not, drop it all
                _currgeom = _bigger.union_all()
    
    elif _available['combined_total_votes'].sum() - _available.loc[i]['combined_total_votes'] < dmin: 
        markillegal(i, "too small without individual")  #if dropping this compromises district size
    
    else: 
        dropprec(i, "")  #if no other issues, drop it like its hot!
        _currgeom = _geomin
  {% endhighlight %}





  Putting it all together, below is the complete process.  The loop is also nested to repeat for both Democratic districts:

  {% highlight python %}
#define functions used in the loops:
def pcttarget(gdf):  #def for ease
    return gdf['total_'+target_party].sum() / gdf['combined_total_votes'].sum()

def markillegal(idx, note):
    _illegal.append(idx)
    log.append([d, _c, idx, 'illegal', note])
    
def dropprec(idx, note):
    if idx not in _available.index: return #possible precs were dropped through brnach interactions, don't error out
    _available.drop(idx, inplace=True)  #drop it from the df  #must use inplace modifiers from within this function
    _illegal.clear()  #wipe illegal as geom has changed (i know size ones could be left in here... but simplicity and we aren't going to hit that)
    log.append([d, _c, idx, 'dropped', note])

def add2district(idx, note):
    if idx not in _available.index: return
    _district.append(idx)
    _available.drop(idx, inplace=True)
    _illegal.clear()
    log.append([d, _c, idx, 'added', note])


target = 0.27
log = []  #items housed in the log will be of the format [district, iteration, index, "dropped/illegal", notes]

for d in range(1,3): #districts 1-2
    _available = optimized_districts[optimized_districts['district']==0].copy()  #initially, everything is an option for d1
    _currgeom = _available.union_all()  #this will permanently house the unioned geometry (to avoid recalc)
    _startinggeom = _currgeom  #will use this when we get into calculating remaining geometry
    _availstarting = _available.copy()  #hold a copy of what started available (don't risk adding back in already assigned precs)
    
    _illegal = []  #this will house precs that can't be dropped right now (break contiguity)
    _unifyremaining = False  #at the beginning, allow the remaining area to be disjointed (quickly close in on target area), however start reconciling it when we near a solution
    
    print(f"[{datetime.datetime.now():%I:%M:%S %p}] start")
    
    _c = 1  #count loops
    with warnings.catch_warnings():
        warnings.simplefilter("ignore")  #ignore warnings in this section... 
     
        while not (_available['combined_total_votes'].sum() <= dmax and pcttarget(_available) <= target):
            _border = _currgeom.boundary
            _touchmask = _available.geometry.intersects(_border)
            _touchmask2 = not _unifyremaining or _available.geometry.intersects(_remaingeom) #short-circuit this to true before unify turned on, when on, saves unneeded checks
            _illegalmask = ~_available.index.isin(_illegal)
            
            
            _edgeprecs = _available[_touchmask & _touchmask2 & _illegalmask]  #available edge pieces
        
            if len(_edgeprecs)==0:  #if there's nothing left to legally drop
                print(f"no solution found ({_c-1} iterations)")
                break
        
            #use a similar scoring system to r to balance branchyness and reduction
            _sortscores = [] #[idx, score, geomin]
            for i in _edgeprecs.index:
                _igeom = _available.loc[i].geometry
                _geomin = _currgeom.difference(_igeom)
                _branchyness = _geomin.length**2 / _geomin.area

                _t = 30e3 * (_available['combined_total_votes'].sum()-323e3) / 4e6  #lookahead distance shrinks as it gets smaller (starts ~4.5M votes)
                minx, miny, maxx, maxy = _igeom.bounds
                _fastbuffer = (minx-_t, miny-_t, maxx+_t, maxy+_t)  #apparently much faster than .buffer(10e3)
    
                _touchi = _available.iloc[_available.sindex.intersection(_fastbuffer)]  #do this touchi thing again
                _ipct, _touchipct = pcttarget(_available.drop(i)), pcttarget(_available.drop(_touchi.index))
                _ipctscore, _touchipctscore = abs(_ipct - target)**2*1e4, abs(_touchipct - target)**2*1e4
                
                _sortscore = _branchyness + _ipctscore + 0.5*_touchipctscore
                _sortscores.append([i, _sortscore, _igeom])
            
            _sortscores = sorted(_sortscores, key=lambda x: x[1])  #lowest score at the top - these are ones where dropping produces a better (lower score) result
            _ndrop = max(int(len(_edgeprecs)*0.33), 1) #try some speed, drop x% of border or 1
        
            for l in _sortscores[:_ndrop]:    
                i = l[0]
                if l[0] not in _available.index: continue  #in case idx dropped in branch, skip all 
                
                _igeom = l[2]
                _geomin = _currgeom.difference(_igeom)  #**need to recalculate this!  with multiple dropped since scoring, it's out of date
                if isinstance(_geomin, MultiPolygon): _validgeoms = [g for g in _geomin.geoms if g.area>23e3]  #some individual precs genuinely have slivers

                if _unifyremaining: #only calculate this when needed
                    _geomout = _remaingeom.union(_igeom)  
                    if isinstance(_geomout, MultiPolygon): _validout = [g for g in _geomout.geoms if g.area>23e3]  #same deal
                
                #----------legality checks:---------------------------
                if _unifyremaining and isinstance(_geomout, MultiPolygon) and len(_validout)>1: markillegal(i, "fractures remaining")  #this one only gets checked at the end
                
                elif isinstance(_geomin, MultiPolygon) and len(_validgeoms)>1:  #if dropping this produces a multipolygon (ie fractures the contiguity)... don't immediately say the drop is illegal...
                    if len(_validgeoms)>2: markillegal(i, "3+ polygons")  #don't bother with the complexity of 3+ polygons
                    else:
                        _poly1 = _available[_available.geometry.intersects(_validgeoms[0])]  #identify the two district polygons
                        _poly2 = _available[_available.geometry.intersects(_validgeoms[1])]
    
                        if _poly1['combined_total_votes'].sum() > _poly2['combined_total_votes'].sum(): _bigger, _smaller = _poly1, _poly2
                        else: _bigger, _smaller = _poly2, _poly1  #name them "bigger" and "smaller" for ease
    
                        _pctbigger = pcttarget(_bigger)  #pct dem in each for ease
                        _pctsmaller = pcttarget(_smaller)
    
                        #run some checks to decide if i is illegal to remove, or we should gas the whole _smaller branch with it
                        if _bigger['combined_total_votes'].sum() < dmin: markillegal(i, "too small without branch")  #if removing this compromises dmin... can't do it
                        elif _pctsmaller < _pctbigger: markillegal(i, "branch closer to target")  #if the branch is higher % than the rest, leave it (coudl change later)
                        else: 
                            _todrop = [i] + list(_smaller.index)
                            for p in _todrop: dropprec(p, f"dropped with branch {i}")  #if not, drop it all
                            _currgeom = _bigger.union_all()
                
                elif _available['combined_total_votes'].sum() - _available.loc[i]['combined_total_votes'] < dmin: 
                    markillegal(i, "too small without individual")  #if dropping this compromises district size
                
                else: 
                    dropprec(i, "")  #if no other issues, drop it like its hot!
                    _currgeom = _geomin

                #evaluate once at end of whichever clause
                if _unifyremaining: _remaingeom = _startinggeom.difference(_currgeom)  #only as needed

            
            #clear out the mini geom gunk that accumulates
            if isinstance(_currgeom, MultiPolygon): 
                _currgeom = max(_currgeom.geoms, key=lambda g: g.area)
                _illegal.clear()
            if _unifyremaining and isinstance(_remaingeom, MultiPolygon): 
                _remaingeom = max(_remaingeom.geoms, key=lambda g: g.area)
                _illegal.clear()
                    
            
            #check to turn on _unifyremaining
            if _available['combined_total_votes'].sum() < 1e6 and not _unifyremaining: #once size passes x, reconcile the remaining area - for dems, this needs to kick in pretty immediately
                _unifyremaining = True  #turn this on
                _remaingeom = _startinggeom.difference(_currgeom)  #define a remaining geometry
                if isinstance(_remaingeom, MultiPolygon):  #if remaining is fractured, add the small bits back in
                    _geoms = sorted(_remaingeom.geoms, key=lambda g: g.area, reverse=True)
                    for g in _geoms[1:]:  #all but the biggest
                        _affected = _availstarting[_availstarting.geometry.intersects(g)]
                        _affected = _affected[~_affected.index.isin(_available.index)]  #don't re-add duplicates
                        _available = pd.concat([_available, _affected])  #put them back into available
                        log.extend([[d, _c, idx, "put back", "reconcile remaining geom"] for idx in _affected.index])
                    _currgeom = _available.union_all()  #reset this
                    _remaingeom = _geoms[0]  #the biggest is all that's left
                    _illegal.clear()  #we aren't calling drop so this doesn't happen automatically
                    
            if _c%10==0 or _c==1:    
                np.savez_compressed("loserlog.npz", log = np.array(log))  #back up the log and current state
                print(f"[{datetime.datetime.now():%I:%M:%S %p}]",
                      f"Iteration {_c}: {np.sum([l[3]=='dropped' and l[1]==_c and l[0]==d for l in log])}/{_ndrop} dropped.",
                      f"Current: {len(_available):,} precs - {_available['combined_total_votes'].sum():,.0f} votes -",
                      f"{pcttarget(_available):.2%} {target_party}."
                 )
            _c+=1
            
    
    
    print(f"finished d{d} - {_available['combined_total_votes'].sum():,.0f} votes, {pcttarget(_available):.2%} {target_party}")
    optimized_districts.loc[_available.index, 'district'] = d  #set district to d for this found region
  {% endhighlight %}
  
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

![]({{ site.baseurl }}/assets/projects/20250910_gerrymandering/winnerdistricts-ezgif.com-speed.gif)
<figcaption>Construction of the remaining 11 Republican districts</figcaption>

![]({{ site.baseurl }}/assets/projects/20250910_gerrymandering/beforeswapresults.png)
<figcaption>2016 NC House election results using the 13 districts above</figcaption>


<details>
  <summary>Full code and method for nerds like me</summary>
  The logic here was more or less the same as "packing", just in reverse (adding, not dropping).  Because the code is so similar I won't go through each piece of it again.  Some notable quick changes: Because we are adding not dropping, there's no risk that adding a neighbouring precinct will break <em>internal</em> contiguity - so the check is only on external.  To minimize problems later on, "branchyness" was also calculated and scored for the <em>remaining</em> geometry.  Because the area we're working with of a single district is so much smaller, the initial scoring was no longer the bottleneck - therefore only the one top-scored option was dropped.  To save runtime on expensive geometry evaluations in legality checks, some of the objects calculated in the scoring stage were stored and passed for reuse in legality evaluation.<br><br>

  This "cracking" stage did add one complexity of where to <em>start</em> each district.  I played around with a few options, including initially seeding intentionally difficult areas to try to get them out of the way while more options were available.  In the end, the addition of the final cleanup step (spoilers) meant the most important thing for the seeds was really just to avoid forcing awkward geometries by seeding in the middle of the map.  The seeds simply ping-pong from the left and right extremes (later geometries were more awkward if all seeded from the same direction):

  {% highlight python %}
target = 0.55  #set this to what min will be in cleanup
log = [] #items housed in the log will be of the format [district, iteration, index, "dropped/illegal", note]

with warnings.catch_warnings():
    warnings.simplefilter("ignore")  #warnings galore lol

    
    for d in range(3,14):  #districts 3-13    
        _available = optimized_districts[optimized_districts['district']==0].copy()
        
        #update min and max relative to what's left:
        _votesleft = _available['combined_total_votes'].sum()
        _buffer = (1-d/13)*30e3 #the idea is to start with tighter guardrails on allowable sizes, that open up to 0 by the end
        max_size = min(dmax, _votesleft - (13-d)*(dmin+_buffer))  #need to leave at LEAST enough for remaining dists to be able to reach dmin (w10% buffer)
        min_size = max(dmin, _votesleft - (13-d)*(dmax-_buffer))  #can't leave so many later dists must exceed dmax
        if d==13: min_size = max_size = _votesleft #except the last time, have to take everything
        if min_size > max_size:
            print("no valid solution possible")
            break 
    
        #set up empty variables for new district:
        _district = []  #will house the idx of this district's pieces
        _illegal = []  #same approach as before
        _emergencyforcebranch = False  
        _currgeom = GeometryCollection()  #empty geometry object to hold _district union
        _availgeom = _available.union_all()  #starts with everything available
        _c = 1
    
        #define a seed for this district:
        print(f"[{datetime.datetime.now():%I:%M:%S %p}] building D{d} seed...")
        
        if d in range(0,0):  #optionally - for the first n, try to get the hard stuff out of the way
            #for each p in _available, find the most difficult 100k votes surrounding it
            _seedpcts = []  #[pct, indexes, geomin, geomout]
            
            for _i, p in _available.sample(frac=0.05, random_state=69).iterrows():  #for speed, only consider 1/10 as candidates for seed
                _check = _available.loc[[_i]]
                _checksize = 0
                while _checksize < 200e3:  #go the place with the hardest 200, but only force seed the first 100                
                    _checkgeom = _check.union_all()
                    if _checksize <= 100e3: 
                        _seed = _check  #only store it as the true seed under 100k
                        _seedgeom = _checkgeom  #save these, avoid re-evaluating
                    _prefilter = _available.iloc[_available.sindex.intersection(_checkgeom.bounds)]
                    _check = _prefilter.loc[_prefilter.intersects(_checkgeom)]  #iteratively expand out to touching areas
                    _checksize = _check['combined_total_votes'].sum()
                
                _seedavailgeom = _availgeom.difference(_seedgeom)
                
                if isinstance(_seedgeom, Polygon) and isinstance(_seedavailgeom, Polygon):  #only add an an option if it doesn't break continuity
                    _pctdiff = abs(pcttarget(_seed) - target)
                    _seedpcts.append([_pctdiff, _seed.index, _seedgeom, _seedavailgeom])  #save the geoms for reuse
            
            _worstseed = max(_seedpcts, key=lambda x: x[0])  #start with the FURTHEST to go
            for s in _worstseed[1]: add2district(s, "seed") #item 1 is a list of indexes
            _seedgeom, _seedavailgeom = _worstseed[2], _worstseed[3]  #recover these to avoid recalc
                
            _currgeom = _seedgeom
            _availgeom = _seedavailgeom
            print(f"D{d} seed: {optimized_districts.loc[_district]['combined_total_votes'].sum():,.0f} votes - {pcttarget(optimized_districts.loc[_district]):.2%} {target_party}")

        else: #left,right,left,right - better for contiguity at the end
            if d%2==0: _seedidx = _available.geometry.bounds['maxx'].idxmax()
            else: _seedidx = _available.geometry.bounds['minx'].idxmin()
            add2district(_seedidx, "seed")
            _currgeom = optimized_districts.loc[_seedidx].geometry
            _availgeom = _availgeom.difference(_currgeom)
        
        print(f"[{datetime.datetime.now():%I:%M:%S %p}] D{d} start [{min_size/1e3:.0f}k - {max_size/1e3:.0f}k]")
    
        #####################LOOP###################################
        
        while not (min_size <= optimized_districts.loc[_district]['combined_total_votes'].sum() <= max_size  and  abs(pcttarget(optimized_districts.loc[_district])-target) <= 0.02):  #consider success within x% of target
            #define options:
            _illegalmask = ~_available.index.isin(_illegal)
            _touchmask = _available.geometry.intersects(_currgeom)  #touches the current district
            _options = _available[_illegalmask & _touchmask].index  #available options for precs - shockingly we only ever need the indexes
            if len(_options) == 0:
                if _emergencyforcebranch == False:  #turn on emergency mode and reset the current loop before full abort
                    _emergencyforcebranch = True
                    _illegal.clear()
                    continue
                else:
                    print("no solution found")
                    break
    
            #sort a priority score for finding best next addition:
            _sortscores = []  #[index, score, geomin, geomout]
            for i in _options:
                _igeom = _available.loc[i].geometry
                _geomin = _currgeom.union(_igeom)  #the inside geometry with this added
                _geomout = _availgeom.difference(_igeom)  #the surrounding geometry withOUT this prec too
                _branchynessin = _geomin.length**2 / max(_geomin.area,0.1)  #the "branchiness" (permieter sq over area)  #max(,0.1) for safe division on the last rep
                _branchynessout = _geomout.length**2 / max(_geomout.area,0.1)
                
                #be a little more intelligent on seeking percentages than just the prec itself... sindex everything it touches (find a smart DIRECTION to expand)
                _wi = pd.concat([optimized_districts.loc[_district], _available.loc[[i]]])
                _touchi = _available.iloc[_available.sindex.intersection(_igeom.buffer(10e3).bounds)]
                _wtouchi = pd.concat([optimized_districts.loc[_district], _touchi])
                
                _touchipct, _ipct = pcttarget(_wtouchi), pcttarget(_wi)
                _touchipctscore, _ipctscore = abs(_touchipct-target)**2*1e4, abs(_ipct-target)**2*1e4  #(1%=1, 2%=4, 5%=25)
    
                _sortscore = _branchynessin + _branchynessout + _ipctscore + 0.5*_touchipctscore  #num is just a bit arbitrarily picked as a weighting to combat typical branchiness (~80-200)
                _sortscores.append([i, _sortscore, _geomin, _geomout])  #save these to pass in later (avoid recalc)
                    
            _sortscores = sorted(_sortscores, key=lambda x: x[1])  #sort on ascending score (lowest first)
    
            #check options in priority order for validity:
            for l in _sortscores:
                #unpack the goodies we already calculated during scoring
                i = l[0]  #index
                if i not in _available.index: continue  #possible it was dropped as a branch
                
                _option = _available.loc[i]
                _geomin = l[2]  #we already calculated geomin as part of the scoring process
                _geomout = l[3]  #and same for out

                if isinstance(_geomout, MultiPolygon): _validgeoms = [g for g in _geomout.geoms if g.area>23e3]  #get rid of slivers
                
                if isinstance(_geomout, MultiPolygon) and len(_validgeoms)>1: 
                    if len(_validgeoms) > 2: markillegal(i, "remaining multipoly")  #if there are 2+ polygons, can't do this
                    else: 
                        #don't immediately throw away branches... could take the whole branch
                        _smallgeom = min(_validgeoms, key=lambda g: g.area)  #the smaller poly that would be left
                        
                        _affected = pd.concat([
                            _available[_available.geometry.intersects(_smallgeom)],
                            _available.loc[[i]]
                        ]) #these are the precs included in the branch
                        _affectedsize = _affected['combined_total_votes'].sum()  #votes this branch would add
                        _affectedpct = pcttarget(_affected)  #%makeup of the branch
                        _affgeom = _affected.union_all()
                        
                        _currsize = optimized_districts.loc[_district]['combined_total_votes'].sum()
                        _currpct = pcttarget(optimized_districts.loc[_district])
                        _newpct =  (_currpct*_currsize + _affectedpct*_affectedsize) / (_currsize + _affectedsize) #pct the district would be with this added
                        
                        _sizecheck = _currsize + _affectedsize < max_size  #adding this doesn't ruin size
                        _damagecheck = abs(_newpct - target) <= abs(_currpct - target) + 0.01  #adding this helps (or doesn't really hurt)
                        
                        if _sizecheck and (_damagecheck or _emergencyforcebranch):  #if we productively add this, or if it's either this or abort...
                            for _add in _affected.index: add2district(_add, f"forced branch from {i}")  #add all
                            _currgeom = _currgeom.union(_affgeom)
                            _availgeom = _availgeom.difference(_affgeom)
                            _emergencyforcebranch = False  #then turn it back off to try again
                            break #don't evaluate non-branch related clauses below 
                        else: markillegal(i, "cannot add required branch")
                
                elif optimized_districts.loc[_district, 'combined_total_votes'].sum() + _option['combined_total_votes'] > max_size: markillegal(i, "district too big")
                
                else: 
                    add2district(i, "")
                    _currgeom = _geomin  #can reuse these
                    _availgeom = _geomout
                    break  #don't do the rest of the options 
    
            #print & back up results:
            if _c%50==0 or _c==1:
                print(f"[{datetime.datetime.now():%I:%M:%S %p}]",
                      f"D{d}-iteration {_c}: {len(_district)} precs in district -",
                      f"{optimized_districts.loc[_district]['combined_total_votes'].sum():,.0f} votes -", 
                      f"{pcttarget(optimized_districts.loc[_district]):.2%} {target_party}"
                     )
                np.savez_compressed("winnerlog.npz", log = np.array(log))  #back up the log and current state
            
            _c+=1
    
        #end of district cleanup:
        if d==13: #dump everything at the end (float rounding can make it 0.000001 "too big")
            for p in _available.index: add2district(p, "end of run dump")  
        
        np.savez_compressed("winnerlog.npz", log = np.array(log))  #one last backup
        optimized_districts.loc[_district, 'district'] = d  #assign
        
        print(f"finished district {d}: {optimized_districts.loc[_district]['combined_total_votes'].sum():,.0f} votes -", 
              f"{pcttarget(optimized_districts.loc[_district]):.2%} {target_party}"
             )
  {% endhighlight %}
</details>









### Attempt 2.5: Monte Carlo Again??
After probably far too long of beating my head against the wall trying to get the first pass above to be perfect, I decided to revisit our old friend Monte Carlo.  I realized that the target construction above was getting very close, and that the easiest thing might just be to employ a more targeted swapping approach to "clean up" the districts and get it across the line.  The full method is below, but the high-level approach was to idenitfy swaps that were *mutually beneficial* to the overall map, with the objective of getting all Republican districts to at least a 55% majority:

![]({{ site.baseurl }}/assets/projects/20250910_gerrymandering/swapcleanup-ezgif.com-speed.gif)
<figcaption>A final "swapping" cleanup on the district map to balance Republican margins</figcaption>

<details>
  <summary>Full code for nerds</summary>
  For this process, the idea was to simply identify all potential "swaps" that were mutually beneficial to the districts.  Similar to above, my definition of "mutually beneficial" was based on a net increase in the combined scoring metric that was used throughout the previous steps<br><br>

  The script goes district by district (biggest first, to maximize available options), identifies precincts that sit on its border with another district, and evaluates the combined score impact to both districts if that precinct were to switch.  If the overall score is better, similar legality checks are performed and then the precinct is off to its forever-home (or at least, for-this-iteration-home).  I didn't bother with marking things illegal here since the number of iterations was quite small.<br><br>

  In order to ensure a solution is found, the relative weights of demographic makeup and "branchyness" are dynamic here.  Initially they start equally weighted as above, however if all mututally beneficial options are exhausted, the relative weighting on demographic makeup is increased.  In this way, the script will first exhaust all "good looking" options, and then resort to more and more absurd-looking geometries until it is able to adequately balance voter demographics:

  {% highlight python %}
#quick function for "scoring" district configurations - changing to take geom and pct inputs... too slow
def scoredistrict(pct, geom):
    branchyness = geom.length**2 / geom.area
    pctscore = abs(pct-target)**2 * 1e4

    return branchyness + pctwgt*pctscore  #crank the pct score


    gdfr = optimized_districts[optimized_districts['district']>=3].to_crs("Web Mercator").copy()  #only do this for the majority ones

target = 0.57  #set this to true target
minimum = 0.55
log = []  #[iteration, idx, from, to]
pctwgt = 1  #this will crank up until we find a solution

#initialize district scores based on criteria used previously - holding everything in static tables for speed
distszs = gdfr.groupby('district')['combined_total_votes'].sum()
distpcts = gdfr.groupby('district').apply(pcttarget, include_groups=False)  #static this because I call it all the time
distgeoms = gdfr.dissolve('district')['geometry'] #geometry of each unioned district
distscores = pd.Series([scoredistrict(p, g) for p, g in zip(distpcts, distgeoms)], index = distpcts.index)

print(distpcts)
print(f"[{datetime.datetime.now():%I:%M:%S %p}] start")

with warnings.catch_warnings():
    warnings.simplefilter("ignore")  #negative buffer warnings
    
    _c=1
    _swapct = 1
    while min(distpcts) < minimum:
        if _swapct==0: #if no swaps were made last loop, double the pct weighting
            pctwgt*=2
            distscores = pd.Series([scoredistrict(p, g) for p, g in zip(distpcts, distgeoms)], index = distpcts.index)  #recalculate existing scores
        
        _checkct=0
        _swapct=0
        
        _dorder = gdfr.groupby('district')['combined_total_votes'].sum().sort_values(ascending=False).index  #go through districts biggest to smallest
        for d in _dorder:  
            _district = gdfr[gdfr['district']==d]
            _border = _district.union_all().boundary
            _edgeprecs = _district[_district.geometry.intersects(_border)]
    
            _nondistrict = gdfr[gdfr['district']!=d]
            _swappable = gpd.sjoin(_edgeprecs, _nondistrict)  #intersects/inner is default predicate
            for _i, r in _swappable.iterrows():  #check for a valid donation from left -> right            
                #if _checkct%250==0: print(f"[{datetime.datetime.now():%I:%M:%S %p}] {_swapct}/{_checkct} swaps...")
                _checkct+=1
                
                _dleft, _dright = r['district_left'], r['district_right']
                
                _igeom = r['geometry']
                _isz = r['combined_total_votes_left']  #left is the actual p from edgeprecs
                _ipct = r['pct_'+target_party+'_left']
                
                if gdfr.loc[_i, 'district'] != _dleft: continue  #possible this was already swapped, some intersect with many                

                #calculate scores if this moved (from math, avoid geom)
                _leftnewsz = distszs[_dleft] - _isz
                _leftnewpct = (distpcts[_dleft]*distszs[_dleft] - _ipct*_isz) / _leftnewsz
                _leftnewgeom = distgeoms[_dleft].difference(_igeom)
                _leftnewscore = scoredistrict(_leftnewpct, _leftnewgeom)

                _rightnewsz = distszs[_dright] + _isz
                _rightnewpct = (distpcts[_dright]*distszs[_dright] + _ipct*_isz) / _rightnewsz
                _rightnewgeom = distgeoms[_dright].union(_igeom)
                _rightnewscore = scoredistrict(_rightnewpct, _rightnewgeom)

                #validity checks
                _szcheck = _leftnewsz >= dmin and _rightnewsz <= dmax  #donation doesn't break either
                _contcheck = isinstance(_leftnewgeom, Polygon) and isinstance(_rightnewgeom, Polygon)  #contiguity preserved
                _scorecheck = _leftnewscore + _rightnewscore < distscores.loc[[_dleft, _dright]].sum()  #collectively, scores are lower than they were

                if _szcheck and _contcheck and _scorecheck:
                    gdfr.loc[_i, 'district'] = _dright  #switch districts
                    _swapct+=1
                    log.append([_c, _i, _dleft, _dright])
                    distscores.loc[[_dleft, _dright]] = [_leftnewscore, _rightnewscore]  #move these scores in as the new baselines
                    distpcts.loc[[_dleft, _dright]] = [_leftnewpct, _rightnewpct]  #move these scores in as the new baselines
                    distszs.loc[[_dleft, _dright]] = [_leftnewsz, _rightnewsz]  #move these scores in as the new baselines
                    distgeoms.loc[[_dleft, _dright]] = [_leftnewgeom, _rightnewgeom]  #move these scores in as the new baselines
                    
    
        np.savez_compressed("pctcleanuplog.npz", log = np.array(log))  #backup results
        optimized_districts.loc[gdfr.index, 'district'] = gdfr['district']  #move into optimized districts
        with open("pctclean_midrun.pkl", 'wb') as f: pickle.dump(optimized_districts, f)  #back this up to keep an eye on mid run
        
        print(f"[{datetime.datetime.now():%I:%M:%S %p}] iteration {_c} [wt={pctwgt}]:",
              f"{_swapct}/{_checkct} swaps (min = {min(distpcts):.3%})"
             )
        _c+=1


print(f"[{datetime.datetime.now():%I:%M:%S %p}] {'no swaps remain' if _swapct==0 else 'success'}") 
print(distpcts)
  {% endhighlight %}
</details>





And with that, we have officially achieved the "impossible" 11-2 Republican split!  Below is the final district mapping and district election results for the 2016 NC House election:

<iframe src="{{ site.baseurl }}/assets/projects/20250910_gerrymandering/finalmap.html" height="400" width="100%"></iframe>
<figcaption>The final optimized district mapping to achieve an 11-2 Republican split in the 2016 NC House election</figcaption>

![]({{ site.baseurl }}/assets/projects/20250910_gerrymandering/finalmapresults.png)
<figcaption>2016 House election results by NC district using the optimized mapping - Republicans win in 11 of 13 districts with a margin or 55% or higher in each</figcaption>







## Final Thoughts















