---
layout: single
title: "Toronto Bicycle Theft"
date: 2025-08-11
subtitle: 'Analyzing historical bicycle theft in Toronto and predicting correlated geographic features'
excerpt: 'Analyzing historical bicycle theft in Toronto and predicting correlated geographic features'
author_profile: false
tags: [Geospatial Analysis, Machine Learning, Deep Learning]
header:
  teaser: /assets/projects/20250811_biketheft_assets/teasermap.png
---

*tl;dr: Here's where to and not to park your bike in Toronto!  Read on for a full description of the analysis, including identifying key geographic features correlated with high theft*

<iframe src="{{ site.baseurl }}//assets/projects/20250811_biketheft_assets/foliumHmapRaster400.html" height="400" width="100%"></iframe>
<figcaption>Heatmap of relative police-reported Toronto bike thefts between 2014-2025 (400m granularity)</figcaption>




# Intro
Just a fun little one!  In 2009, the City of Toronto launched its Open Data Portal - which exposes several key operational datasets from municipal organizations to the public.  In perusing this portal, a dataset that caught my eye was a record of all TPS-reported bicycle thefts in Toronto from 2014 to present.  As an avid cyclist and proud owner of a 1990s marketplace bargain bike, I thought it would be fun to take a look and see if I could identify any insights to help protect my trusty steed.

# Getting the Data
This project started with the cart a bit in front of the horse, as the catalyst was the dataset and I was trying to figure out what we could do with it.  TPS's Bicycle Theft database contains an entry for each of the over 38,000 reports filed since 2014 reporting stolen bicycles.  It includes date, time, and location features for both the incident and report filing, sparsely-populated information on bicycle make, model, and cost (to be fair I couldn't tell you what make my bike is... i just say "yellow"), and status updates for the rare instances when a bike is actually recovered.

I wanted to enhance the dataset a bit, so I passed a subset of recent data through Nominatim - a free open-sourced API for identifying address information based on coordinates.  Nominatim is built on top of OpenStreetMap (OSM - basically a free open-sourced Google Maps database), so I also downloaded a full OSM data dump of Toronto geographic features (parks, restaurants, bars, etc.) to see if I could identify any correlation between theft and the surrounding area.  Put it all together and we're off to the races!

<details>
  <summary>Full code for API nerds</summary>
  For once, the core dataset itself was just a simple plug and play!

  {% highlight python %}
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns

data = pd.read_csv("bicycle-thefts - 4326.csv")

stolen = data[data['STATUS'] == 'STOLEN']
stolen = stolen.dropna(subset = ['LAT_WGS84', 'LONG_WGS84'])  #drop without location (only 30)
stolen = stolen.drop_duplicates(subset=['EVENT_UNIQUE_ID'], keep='last')  #for some reason, there are duplicate lines for some unique records...
  {% endhighlight %}




  Quick calling of recent data through Nominatim.  Note that this code includes an interrupter to limit the speed and volume of data sent to Nominatim at once (in correspondence with the API's terms of service), as well as logging to avoid any unnecessary re-runs or load on the server:

  {% highlight python %}
#declare a subset to be passed through nominatim
subset = stolen[stolen['OCC_YEAR'] >= 2024]  #the subset we will pass through nominatim

try: 
    with open("NominatimLocations.json", 'r') as f:  #call the backup if we already ran this
        locations = json.load(f)
except:
    locations = {}  #empty dict to house nominatim returns

to_run = subset[~subset['EVENT_UNIQUE_ID'].isin(locations.keys())]  #avoid re-running things we already have (~ = "not")
to_run.reset_index(drop=True, inplace=True)  #reset the index so we can use it as a counter with iterrows

for _, r in to_run.iterrows():
    _id = r['EVENT_UNIQUE_ID']
    lat = r['LAT_WGS84']
    long = r['LONG_WGS84']
    nominatim = Nominatim(user_agent="Personal portfolio work! (hayden.estabrook@gmail.com)").reverse([lat, long])  #the service wants a user_agent for tracking

    locations[_id] = {
        'address_dict': nominatim.raw.get('address', {}),  #the {} is just the "else" clause, returns empty dict to avoid downstream errors from missing addresses
        'lat': nominatim.latitude,
        'long': nominatim.longitude
    }
    
    time.sleep(2.0 + random.uniform(-0.5,0.5))  #API ToS limits to one request per s (jitter, try to avoid auto-lock)

    if _%200 == 0: #stop every ~5mins and backup, and pause to avoid server hammering
        with open("NominatimLocations.json", 'w') as f:
            json.dump(locations, f)

        print(f"{_:.0f} / {len(to_run):.0f} complete.")
        
        if _>0:  #don't sleep after the first loop
            time.sleep(random.uniform(5,10)*60)

#final dump
with open('NominatimLocations.json', 'w') as f: json.dump(locations, f)
  

#insert nominatim data back
stolen2 = subset.copy()  #.copy() forces a version to be loaded to memory not a view

for _i, r in stolen2.iterrows():
    try:
        ldict = locations[r['EVENT_UNIQUE_ID']]
        addr = ldict['address_dict']  #sub dictionary of address fields

        for key, val in addr.items():
            stolen2.loc[_i, key] = val  #add each element of the dictionary (#, street, postal, etc.) to a column
    except:
        continue #nothing

stolen2['postcode_first3'] = stolen2['postcode'].str[:3]
  
  {% endhighlight %}




  OSM geographic features data is imported, but held separately for the predictive model later on:

  {% highlight python %}
#I would say the useful OSM pieces are buildings, natural (poly), points (pt), railways, roads, waterways (ln)
shpfiles = ['buildings', 'natural', 'points', 'railways', 'roads', 'waterways']

src_crs = gpd.read_file(f"OSMToronto_shp/{shpfiles[0]}.shp").crs  #identify source crs

features = gpd.GeoDataFrame(crs=src_crs, columns=['geometry'])  #start blank

for i in shpfiles:
    gdf = gpd.read_file(f"OSMToronto_shp/{i}.shp")
    gdf['source'] = i
    if gdf.crs == features.crs:  #confirm crs matches
        features = pd.concat([features, gdf], ignore_index=True)

features = features.to_crs(epsg=32617)  #convert to a CRS that support meters

#a very small handful have invalid geometries... clean up (but only polygons)
features['geometry'] = features.make_valid()

features=features[features['geometry'].is_valid]  #drop invalid geoms
features=features[~features['geometry'].is_empty]  #and empty

import string
features['type'] = features['type'].str.replace("["+string.punctuation+"]", "_", regex=True)  #replace troublesome names (json characters)
  {% endhighlight %}
</details>








# The Analysis

## Theft Dates and Times
The first thing I was interested to see was *when* bicycle theft was happening.  The first heatmap below breaks down bicycle theft season and year.  Unsurprisingly, bike theft is only really an issue in the summer (shocker), and appears to actually be on the decline in recent years:

![]({{ site.baseurl }}/assets/projects/20250811_biketheft_assets/seasonal_hmap.png)
<figcaption>Heatmap of TPS-reported Toronto Bicycle Thefts by year and month</figcaption>



<details>
  <summary>Full code for nerds</summary>
  Suuuuch a simple pivot table:

  {% highlight python %}
heatmap1 = stolen.pivot_table(index = 'OCC_YEAR', columns='OCC_MONTH', values='EVENT_UNIQUE_ID', aggfunc='size')

#sort values
import calendar
heatmap1 = heatmap1[list(calendar.month_name)[1:]].sort_index()  #list out columns in order to specify value, being lazy and using the calendar module's list 

plt.figure(figsize=(10,5))
sns.heatmap(heatmap1, cmap="coolwarm", annot=True, fmt='.0f', annot_kws = {'size':7})
plt.xlabel("Month")
plt.ylabel("Year")
plt.title("Toronto Bike Theft by Season")
  {% endhighlight %}
</details>






Additionally, I was also curious to look at specific times of day.  The heatmap below shows average bicycle thefts by hour and weekday.  Interestingly, there appears to be a strong trend towards theft just outside of typical working hours (morning commute, lunchtime, and right after work).  

I had initially assumed that this was just a lag in reporting (i.e. an office worker walks outside for lunch to realize their bike was stolen, and doesn't know the exact time it happened), but the trend actually completely disappears in *reported* times!  In fact, most bike thefts are not reported for **1-2** days after the incident.  After a bit of research, the explanation seems to be that theft happens when people leave their bike unattended for "just a minute" to pop in and get lunch or stop on their commute home.  Apparently many will wait to report in hopes that the bike shows up, or will wait until their next convenient downtime (apparently on company time):

![]({{ site.baseurl }}/assets/projects/20250811_biketheft_assets/hourly_hmap.png)
<figcaption>Heatmap of TPC-reported bicycle theft by weekday and hour of occurrence</figcaption>

![]({{ site.baseurl }}/assets/projects/20250811_biketheft_assets/reported_hmap.png)
<figcaption>Heatmap of TPC-reported bicycle theft by weekday and hour of <strong><em>reporting</em></strong>.  Notably, previous hourly trends disappear implying they are not an artifact of reporting times.</figcaption>

![]({{ site.baseurl }}/assets/projects/20250811_biketheft_assets/reported_vsocc.png)
<figcaption>Heatmap of average days to report an incident of bicycle theft, based on the believed time of occurrence.  Notably, thefts on Fridays often go unreported until Monday, and reports made 2+ days after the incident appear to not know the exact time of theft (reported as "12:00AM")</figcaption>


<details>
  <summary>Full code for nerds</summary>
  Why did you think this one would be more interesting???

  {% highlight python %}
datadays = (pd.to_datetime(stolen['OCC_DATE'], format='%Y-%m-%d').max() - pd.to_datetime(stolen['OCC_DATE'], format='%Y-%m-%d').min()).days
datawks = datadays*52/365.24

def avg_thefts(x): #need a custom aggfunc
    return x.count() / datawks  #avg net per week in sample

heatmap2 = stolen.pivot_table(index='OCC_HOUR', columns = 'OCC_DOW', values='EVENT_UNIQUE_ID', aggfunc=avg_thefts)
heatmap2 = heatmap2[list(calendar.day_name)].sort_index()

plt.figure(figsize=(10,7))
ax = sns.heatmap(heatmap2, cmap='coolwarm', annot=True, annot_kws={'size':7}, fmt='.2f')
plt.xlabel("Weekday")
plt.ylabel("Time of Day")
ax.set_yticklabels(['12:00 AM', '1:00 AM', '2:00 AM', '3:00 AM', '4:00 AM', '5:00 AM', '6:00 AM', '7:00 AM', '8:00 AM', '9:00 AM', '10:00 AM', '11:00 AM', '12:00 PM', '1:00 PM', '2:00 PM', '3:00 PM', '4:00 PM', '5:00 PM', '6:00 PM', '7:00 PM', '8:00 PM', '9:00 PM', '10:00 PM', '11:00 PM'],
                  rotation = 1)
plt.title("Avg. Toronto Bike Thefts by Hour")


#out of curiosity - try this with REPORTED time
def avg_thefts(x): #need a custom aggfunc
    return x.count() / datawks  #avg net per week in sample

heatmap2 = stolen.pivot_table(index='REPORT_HOUR', columns = 'REPORT_DOW', values='EVENT_UNIQUE_ID', aggfunc=avg_thefts)
heatmap2 = heatmap2[list(calendar.day_name)].sort_index()

plt.figure(figsize=(10,7))
ax = sns.heatmap(heatmap2, cmap='Blues', annot=True, annot_kws={'size':7}, fmt='.1f')
plt.xlabel("Weekday")
plt.ylabel("Time of Day")
ax.set_yticklabels(['12:00 AM', '1:00 AM', '2:00 AM', '3:00 AM', '4:00 AM', '5:00 AM', '6:00 AM', '7:00 AM', '8:00 AM', '9:00 AM', '10:00 AM', '11:00 AM', '12:00 PM', '1:00 PM', '2:00 PM', '3:00 PM', '4:00 PM', '5:00 PM', '6:00 PM', '7:00 PM', '8:00 PM', '9:00 PM', '10:00 PM', '11:00 PM'],
                  rotation = 1)
plt.title("Reported Toronto Bike Thefts by Hour")


#confirm if occ and report time are pretty similar
occvsrpt = pd.DataFrame()
occvsrpt['occ_dttm'] = pd.to_datetime(stolen['OCC_DATE']) + pd.to_timedelta(stolen['OCC_HOUR'], unit='h')
occvsrpt['occ_wkday'] = stolen['OCC_DOW']
occvsrpt['occ_hr'] = occvsrpt['occ_dttm'].dt.strftime('%H:%M')

occvsrpt['rpt_dttm'] = pd.to_datetime(stolen['REPORT_DATE']) + pd.to_timedelta(stolen['REPORT_HOUR'], unit='h')
occvsrpt['rpt_wkday'] = stolen['REPORT_DOW']
occvsrpt['rpt_hr'] = occvsrpt['rpt_dttm'].dt.strftime('%H:%M')

occvsrpt['time_to_report_hr'] = (occvsrpt['rpt_dttm'] - occvsrpt['occ_dttm']).dt.total_seconds() / (60*60)
occvsrpt['time_to_report_d'] = occvsrpt['time_to_report_hr']/24

_pvt = occvsrpt.pivot_table(index='occ_hr', columns='occ_wkday', values='time_to_report_d', aggfunc='median')
_pvt = _pvt[['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday', 'Sunday']] #put int order
plt.figure(figsize=(10,7))
ax = sns.heatmap(_pvt, cmap="Greys", annot=True, fmt='.1f', annot_kws = {'size': 7})
plt.title('Reported Occurrence Time - Days to Report')
ax.set_xlabel("Occurrence Weekday")
ax.set_ylabel("Occurrence Hour")
ax.set_yticklabels(['12:00 AM', '1:00 AM', '2:00 AM', '3:00 AM', '4:00 AM', '5:00 AM', '6:00 AM', '7:00 AM', '8:00 AM', '9:00 AM', '10:00 AM', '11:00 AM', '12:00 PM', '1:00 PM', '2:00 PM', '3:00 PM', '4:00 PM', '5:00 PM', '6:00 PM', '7:00 PM', '8:00 PM', '9:00 PM', '10:00 PM', '11:00 PM'],
                  rotation = 1)
plt.show()
  {% endhighlight %}
</details>







## Theft by Location
Obviously, the most pressing question was *where* bikes were most often stolen in Toronto.  For a first pass at this, I simply plotted the coordinates of each individual theft to reveal a map of Toronto:


![]({{ site.baseurl }}/assets/projects/20250811_biketheft_assets/simple coord map.png)
<figcaption>All TPS-reported incidents of bicycle theft between 2014 and 2025</figcaption>

<details>
  <summary>Full code for nerds</summary>
  This basic plot is just done using matplotlib/seaborn directly off the latitude and longitude for each point.  Later maps will get into Folium for some proper maps, but Folium does not like this many data points (plus, it's not the most valuable way to visualize this anyways):

  {% highlight python %}
plt.figure(figsize=(10,8))
ax = sns.scatterplot(stolen, x = 'LONG_WGS84', y = 'LAT_WGS84', hue = 'OCC_YEAR', s = 2, palette='plasma', alpha=0.6)
ax.legend(title="Year")
  {% endhighlight %}
</details>




This clearly indicates a trend towards the downtown core, but because many points can layer on top of each other, it can be hard to discern relative theft rates for high-theft areas.  To improve on this, I created a heatmap of theft by laying a fishnet grid over the city map and calculating the average annual thefts within each cell.  This makes it much easier to discern the difference in theft rates within the high-theft downtown core, as well as clearly illustrates the individual streets and intersections with high theft:


<iframe src="{{ site.baseurl }}//assets/projects/20250811_biketheft_assets/foliumHmapRaster100.html" height="400" width="100%"></iframe>
<figcaption>Heatmap of relative annual bicycle theft in Toronto from 2014-2025 (100m granularity).  Shoutout to our west-end outlier Dufferin Mall!</figcaption>


<details>
  <summary>Full code and method</summary>
  <em>Now</em> we get the good stuff.  First up was to create a fishnet grid across the city.  In order to make the size of the grid variable, I defined it first at the smallest granularity (50m in this case) so that I could aggregate up into larger cell sizes:

  {% highlight python %}
#create a grid covering the search area
from shapely.geometry import Polygon

xmin, ymin, xmax, ymax = search_area.bounds  #start with the maximum square bounds

cell_size = 50 #meters, per CRS  (starting at our smallest possible grid size, will optimize)
grid_cells = []
xis, yis = [],[]  #empty lists to house row/col indexes

_c=0
for _x, x in enumerate(np.arange(xmin, xmax, cell_size)):
    for _y, y in enumerate(np.arange(ymin, ymax, cell_size)):
        cell = Polygon([  #list the bounding points as tuples in the orderthey will connect (clockwise)
            (x,y),
            (x, y+cell_size),
            (x+cell_size, y+ cell_size),
            (x+cell_size, y)
        ])
        grid_cells.append(cell)
        xis.append(_x)
        yis.append(_y)

        if _c%50000 == 0: print(f"{_c}")
        _c+=1

search_grid = gpd.GeoDataFrame(geometry=grid_cells, crs ='WGS84') #convert to gdf
search_grid['cell_size'] = cell_size
search_grid['cell_area'] = cell_size**2
search_grid['xindex'] = xis
search_grid['yindex'] = yis

#clip to just the portion inside our search area
search_grid = search_grid.clip(search_area)
  {% endhighlight %}



  Next, I calculated the average annual bike thefts within each cell:

  {% highlight python %}
#identify how many thefts in each grid cell

grid_data = search_grid.copy()  #make a copy to house features

datadays = (pd.to_datetime(stolen['OCC_DATE']).max() - pd.to_datetime(stolen['OCC_DATE']).min()).days

_c=0
for _i, r in grid_data.iterrows():
    thefts = stolen_gdf.within(r['geometry']).sum()
    grid_data.loc[_i, 'annual_thefts'] = thefts / (datadays/365)

    if _c%50000 == 0: print(f"{_c}/{len(grid_data)}")
    _c+=1
  {% endhighlight %}



  Finally, I created a dictionary of grids at different sizes (50-1000m) by iteratively aggregating up the base grid:

  {% highlight python %}
#create a dictionary of grid sizes at different granularities
grid_optimize = {}  #overarching dict to hold trail results

for n in range(1, 21): #(50-1000m cells)
    _key = n*cell_size  #use the cell size as dict key
    grid_optimize[_key] = {}  #there will be different things stored for each n
    
    _gdf = grid_data.copy()
    _gdf['agg_x'] = _gdf['xindex'] // n
    _gdf['agg_y'] = _gdf['yindex'] // n

    #dissolve polygons and aggregate values
    _agggdf = _gdf.dissolve(
        by=['agg_x', 'agg_y'],
        aggfunc={'annual_thefts': 'sum', 'cell_area': 'sum'}
    )

    _agggdf['features_found'] = False  #this will come in handy for avoiding re-calculating rows later

    grid_optimize[_key]['search_grid'] = _agggdf.reset_index()

    print(f"{_key}m complete")
  {% endhighlight %}


  Finally, put that sucker on a map!  This data was much too large for Folium to handle directly as a choropleth, so I rasterized the heatmap layer, and then overlaid it onto the map:

  {% highlight python %}
#create a raster image of heatmap
import geopandas as gpd
import datashader as ds
import datashader.transfer_functions as tf

n = 100
plot_data = grid_optimize[n]['search_grid']

scale_max = max(plot_data['annual_thefts'])+1
scale_99 = np.percentile(plot_data['annual_thefts'], 99.5)
scale = np.append(np.linspace(0,scale_99, 14), scale_max)

raster_bounds_utm=plot_data.total_bounds  #plot bounds in current UTM 17 cRS
raster_bounds_wgs = plot_data.to_crs('WGS 84').total_bounds

canvas = ds.Canvas(  #create a canvas
    plot_width=2000,
    plot_height=2000,
    x_range=(raster_bounds_utm[0], raster_bounds_utm[2]),
    y_range=(raster_bounds_utm[1], raster_bounds_utm[3])
)

agg = canvas.polygons(  #
    plot_data,
    geometry="geometry",
    agg=ds.mean("annual_thefts")  #polygons shouldn't overlap in the raster, but an aggregate object is required for tf.shade
)

shaded = tf.shade(agg, cmap=['white', 'bisque', 'orange', 'red', 'darkred'], how='log')  #shade agg object
shaded.to_pil().save(f"HeatmapRaster{n}.png")
shaded


#create a heatmap of the full data grid
m = folium.Map(location=[43.7, -79.4], zoom_start=12, tiles='CartoDB positron', control_scale=True)

folium.raster_layers.ImageOverlay(
    image="HeatmapRaster100.png",
    bounds=[[raster_bounds_wgs[1], raster_bounds_wgs[0]], [raster_bounds_wgs[3], raster_bounds_wgs[2]]],  #lat/lon, not x/y
    opacity=0.5
).add_to(m)

#mark features of interest
_feats = ['']
for _i, r in features[features['type'].isin(_feats)].to_crs('WGS84').iterrows():  #marker expects WGS84
    centroid = r['geometry'].centroid  #in case of polygons
    lat = centroid.y
    long = centroid.x
    
    folium.CircleMarker(
        location=[lat, long],
        popup=r['type'].title() + ": " + (r['name'] or '*Unnamed*'),
        radius = 3,
        weight=1,
        opacity = 0.2,
        color = 'navy'
    ).add_to(m)

m
  {% endhighlight %}
</details>








## Identifying Geographic Indicators of Theft
Finally, the last thing I was interested in was to see if we could train a model to *predict* bicycle theft based on the geographic features present in the surrounding area.  I leveraged the OSM features dataset for this, and by mapping its *over 1M* buildings, businesses, and landscape elements onto the fishnet grid above was able to train a prediction model to look for features whose presence correlated with theft rates across each cell.  There will be a big nerdy dropdown on the training process below, but the outcome was a model that could predict annual thefts at any point in Toronto with **85% accuracy** based on the geographic features in the surrounding 200m!


![]({{ site.baseurl }}/assets/projects/20250811_biketheft_assets/LGBMr2.png)
<figcaption>Testing the model's accuracy to predict bicycle theft at any point in Toronto based solely on the geographic features present within 200m (R2=0.85)</figcaption>


<details>
  <summary>Big nerdy dropdown</summary>
  The core idea of this model was to use each "cell" in the fishnet grid as a sample, and to attempt a regression predicting the average annual bike thefts based on the OSM features present in that cell.  I played around with a few different types of models for this, and ultimately got the best results with gradient boosting (LGBM).  While I didn't love the non-determinism and feature weight opacity, the data is extremely high-dimension and more transparent models failed to identify any valuable trend (e.g. R2=-1.10 for linear regression... worse than the monkeys!!). <br><br>

  Starting from the grid above, the first step was to identify all the OSM features present within (or immediately outside) each cell.  The OSM features are tagged with types (e.g. "restaurant").  Once all OSM features within a cell were identified, the total number per type were calculated.  These were added as "features" to the GeoDataFrame, with values representing the number of features of that type present in each cell:

  {% highlight python %}
#run the full dataset just for optimized cell size
for n in [400]:
    _gdf = grid_optimize[n]['search_grid']  #the full thing

    with warnings.catch_warnings():
        warnings.simplefilter("ignore")  #run without warnings

        _c=0
        _todo = len(_gdf[~_gdf['features_found']])
        for _i, r in _gdf[~_gdf['features_found']].iterrows():  #only run features not yet found
            cell_geom = r['geometry'].buffer(n/2)  #add a buffer of "r"
            candidates = features.iloc[features.sindex.intersection(cell_geom.bounds)]

            nearby = candidates[
                candidates.intersects(cell_geom) |
                candidates.within(cell_geom)
            ]

            grouped = nearby['type'].value_counts()
            for _typ, ct, in grouped.items():
                _gdf.loc[_i, _typ] = ct
                
            _gdf.loc[_i, 'features_found'] = True  #mark that this cell has been handled in this expensive step

            if _c%1000 == 0: print(f"{n}m: finding features {_c}/{_todo}")
            _c+=1

    _gdf = _gdf.fillna(0, inplace=True)  #fill NaNs
    
#repickle it
with open("search_grid_optimize.pkl", 'wb') as f:
    pickle.dump(grid_optimize, f)
  {% endhighlight %}
  


  For each of the ~5,000 400m cells in the map of Toronto, we now know the average number of annual thefts reported within it, as well as the number of each type of OSM feature present!  All that was left now was to train.  The data was divided into feature and target datasets, and a random sampling of data was <em>withheld</em> from the training process so as to fairly test the model's performance on never-before-seen data:

  {% highlight python %}
from lightgbm import LGBMRegressor
from sklearn.model_selection import train_test_split

gdf = grid_optimize[400]['search_grid']
gdff = gdf[gdf['features_found']]  #make sure we're only running on ones that found features
x = gdff.drop(columns=['annual_thefts', 'geometry'])
y = gdff['annual_thefts']

xtrain, xtest, ytrain, ytest = train_test_split(x, y, test_size=0.25, random_state=69)

lgbmodel = LGBMRegressor()
lgbmodel.fit(xtrain, ytrain)

from sklearn.metrics import r2_score, root_mean_squared_error
preds = lgbmodel.predict(xtest)
print(f"\n\nR2={r2_score(ytest, preds):.2f}")
print(f"RMSE={root_mean_squared_error(ytest, preds):.2f}\n\n")

plt.scatter(x=ytest, y=preds, c='royalblue', alpha=0.3, s=2)
plt.plot(ytest, ytest, 'k--', lw=0.5)
plt.xlabel("Actual Annual Thefts")
plt.ylabel("Predicted Annual Thefts")
  {% endhighlight %}


  
  In reality, instead of just guessing at 400m as a good cell size for modelling, I actually ran an elbow method to identify the best one.  For each of the grid sizes above (50-1000m), I ran a small random sample of the map through the entire process above and compared performance.  The results of this are below, which were how I landed on 400m cells as the optimum for regression:

  <img src = "{{ site.baseurl }}/assets/projects/20250811_biketheft_assets/LGBM_elbowmethod.png" width="100%">

  {% highlight python %}
for n in grid_optimize:
    _gdf = grid_optimize[n]['search_grid']
    _gdf_sample = _gdf.sample(frac=0.25, random_state=69)  #for this portion, we're only running a sample    

    with warnings.catch_warnings():
        warnings.simplefilter("ignore")  #run without warnings

        _c=0
        _todo = len(_gdf_sample[~_gdf_sample['features_found']])  #copy the total todo for progress printing
        
        for _i, r in _gdf_sample[~_gdf_sample['features_found']].iterrows():  #only run features not yet found, and only 25% of data!
            cell_geom = r['geometry'].buffer(n/2)  #add a buffer of "r"
            candidates = features.iloc[features.sindex.intersection(cell_geom.bounds)]

            nearby = candidates[
                candidates.intersects(cell_geom) |
                candidates.within(cell_geom)
            ]

            grouped = nearby['type'].value_counts()
            for _typ, ct, in grouped.items():  
                _gdf.loc[_i, _typ] = ct  #write values back to the search grid

            _gdf.loc[_i, 'features_found'] = True  #write back to grid so we know we can skip it next time

            if _c%1000 == 0: print(f"{n}m: finding features {_c}/{_todo}")
            _c+=1

    
    _gdff = _gdf.loc[_gdf_sample.index]  #only run lgbm on the sample
    
    _x = _gdff.drop(columns=['annual_thefts', 'geometry'])
    _y = _gdff['annual_thefts']
    _xtrain, _xtest, _ytrain, _ytest = train_test_split(_x, _y, test_size = 0.25, random_state=69)

    print(f"{n}m: testing split complete")

    _lgb = LGBMRegressor(verbose=-1)
    _lgb.fit(_xtrain, _ytrain)    
    _preds = _lgb.predict(_xtest)
    
    #back up results to dict
    grid_optimize[n].update({
        'r2': r2_score(_ytest, _preds),
        'rmse': root_mean_squared_error(_ytest, _preds)
    })
    
    print(f"\n {n}m: \n R2={grid_optimize[n]['r2']:.2f} \n RMSE={grid_optimize[n]['rmse']:.3f}\n\n")


#dump the finished results
with open("search_grid_optimize.pkl", 'wb') as f:
    pickle.dump(grid_optimize, f)
  {% endhighlight %}
</details>






The cool thing that this trained model lets us do as well is to actually start to peak under the hood at which types of geographic features are strong predictors of theft!  Unfortunately, the complexity of the type of model required to give good predictions here means influences are not quite as simple as "2 additional thefts per year for each Walmart in the area".  The impact of any one feature can be non-linear, or may only be significant above a cutoff threshold or in conjunction with *other features*.

That said, the most significant contributors to predictions were **bicycle rental racks** (which have a generally positive relationship with theft), **cafes** (generally positive), **footways and walking paths** (generally *negative*), **apartments** (generally positive), and **pubs** (generally negative).  Again, this model is non-deterministic so please read the full dropdown for analysis and limitations before leaving the bike unlocked during your next pint of Guinness!

![]({{ site.baseurl }}/assets/projects/20250811_biketheft_assets/beeswarm.png)
<figcaption>Beeswarm plot of top SHAP feature contributions to the bicycle theft model.  This shows the 10 most impactful features, and visulaizes the value of each data point versus the direction it pushed the final prediction.</figcaption>


<details>
  <summary>Another big nerdy dropdown</summary>
  Well, if you were nerdy enough to click this I'm going assume you know some basics about LGBM.  Gradient boosting machines like LGBM basically go through a bunch of fancy math to organize the data into a <em>giant</em> decision tree.  These can give super great modelling capacity and adapt to capture multi-feature interactions and non-linear trends - however, the double-edge is that their output is not so interpretable as something like linear regression (with a simple "one number coefficient per feature").<br><br>

  To peak under the hood of LGBM, I used SHAP - which is basically a method drip-feeding real data points through the trained model and watching how the final prediction is moved up or down by each feature.  This is top-down method tells us, for each individual sample, the approximate contribution of a feature to the final prediction.  Because of the complexity of the relationship the model has with each feature, there's no "one number" describing each feature's impact on predictions.<br><br>

  {% highlight python %}
import shap

explainer = shap.TreeExplainer(lgbmodel.booster_)
xvals = xtrain
shap_vals = explainer.shap_values(xvals)

coefdf = pd.DataFrame({
    'Feature': lgbmodel.feature_names_in_,
    'Mean Abs SHAP': np.mean(np.abs(shap_vals), axis=0),
    #'Max Abs SHAP': np.max(np.abs(shap_vals), axis=0),
    #"Hayden Number": np.nanmean((shap_vals / xvals.replace(0, np.nan)), axis=0),  #don't need to handle div/0 because of scaled data... probably still safe
    'Corr': pd.DataFrame(shap_vals, columns=lgbmodel.feature_names_in_).corrwith(pd.DataFrame(xvals, columns=lgbmodel.feature_names_in_), method='spearman')  #convert shap to df for easy corr
})


#top features
display(coefdf.sort_values(by='Mean Abs SHAP', ascending=False).head(20))
  {% endhighlight %}



  The "top features" above were based on mean absolute SHAP score - i.e. the features that caused the model to change its prediction the most overall.  Their directionality (i.e. does presence typically indicates <em>more</em> or <em>less</em> theft) was based on the Spearman correlation between each feature's SHAP scores and raw values.  Basically, "does predicted theft increase when this feature is present" (positive correlation), or "do higher values of this feature lead to <em>lower</em> theft predictions" (negative correlation).<br><br>  
  
  The word "typically" is doing a lot of work in that sentence, as in many cases a feature's impact on the final prediction can have a complicated relationship with its value.  We can see the exact relationship between a feature's value and impact on predictions with dependence plots.  These are scatterplots of each data point comparing the feature's value to the prediction impact it produced.  Because the values of <em>other</em> features can influence a gradient boosting model's interpretation, these plots also introduce coloring showing the value of the next most-correlated feature for each point.  Some examples:<br><br>

  Dependence plot for bicycle rental racks.  A nice simple positive correlation (more bike rentals = more theft in the area).  Note the non-linear, step-wise relationship:
  <img src = "{{ site.baseurl }}/assets/projects/20250811_biketheft_assets/dependence_bikerental.png" width="100%">
  <br>
  Dependence plot for apartments.  Note that the presence of nearby footways <em>decreases</em> predicted theft in cells with identical numbers of apartments:
  <img src = "{{ site.baseurl }}/assets/projects/20250811_biketheft_assets/dependence_apartments.png" width="100%">
  <br>
  Dependence plot for footways.  Notably parabolic:
  <img src = "{{ site.baseurl }}/assets/projects/20250811_biketheft_assets/dependence_footway.png" width="100%">
  <br>
  Dependence plot for pubs.  While there appears to be a positive correlation, the overall Spearman correlation was actually slightly <em>negative</em> (possibly due to the slightly parabolic relationship at high values):
  <img src = "{{ site.baseurl }}/assets/projects/20250811_biketheft_assets/dependence_pub.png" width="100%">
  <br>
  It's worth keeping in mind that each data point in this model has almost <strong>500</strong> features, so the SHAP contributions of any one are a very incomplete picture of how the model actually reached its conclusion about predicted theft in an area.
</details>



## Final Thoughts
That's all for this one!  A very fun quick little analysis, and a surprisingly great opportunity for predictive modelling.  Working with such high-dimensionality data, this was a great opportunity to apply SHAP scores to start untangling the inner workings of a non-deterministic model.  I would classify this project more as "fun data to play with" than "actually valuable finding" as I think there's some pretty serious limitations in the dataset that limit any real applicability:

- **Lack of Control Data**:  While the dataset of stolen bikes is great, what's sorely missing is a dataset of *not-stolen* bikes.  Without the ability to control for theft as a percentage, there's a very fair criticism that what we've really created here is just a map of high-traffic areas.  When we're only trying to predict the absolute number of bicycle thefts, super dangerous low-traffic areas look exactly the same as extremely safe high-traffic ones.  What would actually be useful (and I suspect how most people initially interpret the above) is a map of areas where your bike is *likely* to get stolen - which requires a theft *rate*, not just an absolute number.  I could have started to get at this by incorporating geographic population density, but that seemed like overkill for what this was.<br>We can even see implications of this limitation in the predictive model, where it's effectively just saying "lots of stuff" = "probably more theft".  Bicycle rental racks are prime example of this, since presumably the Toronto Bike Share service was intentional about placing rental racks in high-traffic (specifically, high-*bicycle*-traffic) areas.  Intuitively, I would hypothesize that the presence of rental racks actually has a *negative* impact on theft rates (since less people bring their own bikes), and what we're seeing here is just a very strong correlation with "places people like to lock bikes".<br>I did attempt to measure this weakness by adding a dummy feature to the model for "total number of features".  If this had come out as one of the top predictors then I probably would have just thrown the whole model out.  The fact that this wasn't a strong predictor and the presence of non-linear and multi-feature relationships implies that the model was at least finding *some* valueable insights - but I would take this whole thing with a very sizeable grain of salt.
- **Incomplete Picture of Theft**: Notably, this dataset only includes **police-reported** incidents, which I would hypothesize are a small fraction of total bicycle thefts.  There's a long list of socio-economic and cost-of-bike biases that could influence which incidents riders choose to report, which I suspect are skewing even the relative distribution of theft we see in this data.
- **Broad Categorization of Geographic Features**: While the OSM data is great, its geographic features are quite broadly categorized - which limits the depth of insights we can draw about the true "makeup" of a neighbourhood.  Additional feature metadata (e.g. popularity, peak business times, rating, cost) could reveal a lot of interesting trends that are missed when all establishments are just lumped together as "restaurant".  Similarly, the open-sourced and unstandardized nature of OSM data may introduce biases around which features were chosen to be tagged and how (e.g. is this complex "one apartment" or 100).  That said, I do love working with open-sourced data for exactly these same reasons, since it's very cool to see meaningful high-level trends emerge from such "messy" data when the scale is large enough.


That's all for this time!  Thanks so much for checking this out, and go ride a bike!

Cheers,<br>Hayden






