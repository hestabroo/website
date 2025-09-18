---
layout: single
title: "Spotify Wrapped+"
post_header: true
subtitle: "Creating an easy, interactive tool to analyze *all-time* Spotify listening trends"
excerpt: "Creating an easy, interactive tool to analyze *all-time* Spotify listening trends"
author_profile: false
tags: [Machine Learning, Dashboards]
image: "{{ site.baseurl }}/assets/project_assets/20250826_spotifywrapped/spotifywraped_app_screenshot.png"
date: 2025-08-26
---
![]({{ site.baseurl }}/assets/projects/20250811_spotifywrapped_siteassets/spotifywrapped_app_screenshot.png)
<figcaption>Landing page for the Spotify Wrapped+ app</figcaption>

*tl;dr: Check out [this neat tool](https://haydenestabrook-spotifywrapped.streamlit.app/) I built to analyze your all-time Spotify listening history!  Read on below for an embarassing exposé of my personal music listening.*

## Intro
Music is a huge part of my life, and as someone obsessed with data it should surprise no one that I love Spotify Wrapped season every December.  I love the opportunity to see what my friends have been listening to, discover new artists, and enjoy the embarassing outings of everyone's guilty pleasure songs. 

*However*, I've always felt like the Wrapped summary you get each year is quite surface level, and it felt like a miss given the wealth of data Spotify has that they don't really go beyond "top 10 tracks/artists".  SO, I set out to make a "better" Spotify Wrapped that can go deeper into your personal listening style and look more broadly outside the vacuum of the past 12 months.

## Getting The Data
Spotify has (*had*...spoilers) some great developer APIs to query user play history and track info.  However, the API play history dataset is limited to just the past 50 songs and some basic top tracks/artists info.  

I wanted to build a tool that gave a more comprehensive picture of your listening, so I built the tool around Spotify's "Extended Streaming History", which is an all-time data dump you can request from Spotify.  Unfortunately, this did remove the ability to have an instant sign-on/OAuth UI for the app - but I felt like the pros outweighed the cons here.

The Extended Streaming History comes as a zipped json package, and was honestly pretty plug and play.  Some quick minor data cleanup and helper columns and we were good to go!

<details>
  <summary>Full code for nerds</summary>
  The app is hosted on Streamlit.  Very simple out-of-the-box file uploader and of the zip:
  
  ``` python
    st.write("")
    st.subheader("File Upload")
    zipobj = st.file_uploader(
        "Upload your Spotify Extended Play History (.zip format).  Don't have your play history data yet? [Click here](https://hestabroo.github.io/SpotifyWrapped/SpotifyDownloadInstructions.html) to download it!", 
        type=['zip']
    )
    while zipobj is None:
        st.stop()  #wait until we have a file
    
    st_progress_text = st.empty()
    st_progress_bar = st.progress(0)
    
    #wrap this in a try in case wrong format
    try:
        st_progress_text.write("🤐 Un-zipping your data...")
        with zipfile.ZipFile(zipobj) as z:
            files = [f for f in z.namelist() if f.startswith("Spotify Extended Streaming History/Streaming_History_Audio") and f.endswith(".json")]
            files = sorted(files)
    
            dfs=[]
            for f in files:
                with z.open(f) as fo:
                    data = pd.DataFrame(json.load(fo))
                    dfs.append(data)
    
        streamhx = pd.concat(dfs).reset_index()
        if streamhx.empty: raise BadData("no data imported")  #explicitly call an error if it's empty
    except:
        st.error("Hm... That doesn't seem to be the right file/format... Try again?")
        st_progress_text.empty()
        st_progress_bar.empty()
        st.stop()
  ```

  Besides that, I did some basic cleanup to filter out audiobooks and other lame not-music stuff, as well as to truncate a "tail" at the start of usage (this might have just been a me thing, but my account "existed" a year before I really started using it, so most charts had a year of whitespace).  Also added some QOL columns:
  
  ``` python
    streamhx = streamhx[streamhx['audiobook_title'].isna()]  #remove audiobooks and other nerd shit
    
    streamhx['dttm'] = pd.to_datetime(streamhx['ts'])
    streamhx['dttm_local'] = streamhx['dttm'].dt.tz_convert('America/New_York')  #convert to local timezone
    
    streamhx['year'] = streamhx['dttm'].dt.year
    streamhx['month_start'] = streamhx['dttm'].dt.to_period("M").dt.start_time
    streamhx['week_start'] = streamhx['dttm'].dt.to_period("W").dt.start_time
    streamhx['hour'] = streamhx['dttm'].dt.hour
    streamhx['weekday'] = streamhx['dttm'].dt.day_name()
    
    streamhx['hr_played'] = streamhx['ms_played'] / 1000 / 60 / 60
    
    streamhx.rename(columns={
        'master_metadata_track_name':'song_name',
        'master_metadata_album_artist_name':'artist_name',
        'master_metadata_album_album_name': 'album_name'
    }, inplace=True)  #simplify some column names
    
    start_date = np.percentile(streamhx['dttm'],1)  #exclude tail before really using account...if like me.  should be insignificant otherwise
    streamhx = streamhx[streamhx['dttm']>=start_date]
  ```
  
</details>


## The Analysis
### Top Artists/Tracks
First up was the basics.  Critical as I was of Spotify only really doing the "top 10s", obviosuly it's something everyone want to see, and a pretty cool thing to see across *all time*.  I tried to add a little more info around how many cumulative hours have been spent listening to each track/artist, as well as the "peak listening periods" for each:

![]({{ site.baseurl }}/assets/projects/20250811_spotifywrapped_siteassets/topartists_table.png)
<figcaption>My all-time top artists</figcaption>

![]({{ site.baseurl }}/assets/projects/20250811_spotifywrapped_siteassets/topsongs_table.png)
<figcaption>And my all-time most-listened songs... For the uninitated, The Dirty Nelsons is the band I drum in... embarassinggg</figcaption>

<details>
  <summary>Full code for nerds</summary>
  I don't need to bore you with the basics of finding top artists/tracks.  Only interesting thing here was the identification of a "peak listening window".  A bit arbitrarily, I defined this to be the smallest possible window containing at least 50% of the artist's playtime.<br><br>
  
  At first I tried to do this by iteratively "shrinking" the full date, dropping the lowest-volume end - but this greedy logic got hung up on local peaks.  I ended up looping through each possible window start point and extending <em>outwards</em> until 50% was captured, and finding the best (smallest) window that achieved this:

  ``` python
  #find the smallest possible window containing x% of play time
artist_month.sort_values(by=['artist_name', 'month_start'], inplace=True)
target = 0.5

top_ranges = []
for a in artists.index:  #per artist...
    _df = artist_month[artist_month['artist_name']==a].reset_index()
    _best = {'start': 0, 'end': len(_df)-1, 'size':len(_df), 'pct':1.0}  #initialize the "score to beat" as the whole thing

    for _start in range(len(_df)-1):  #for each possible starting point...
        for _end in range(_start, len(_df)-1):  #slide the window right until...
            _pct = _df['pct_total'][_start:_end+1].sum()
            if _pct >= target:  #...until we capture 50% of volume
                _size = _end-_start
                if _size < _best['size'] or (_size == _best['size'] and _pct > _best['pct']):  #if this is smaller, override best (break ties on higher pct
                    _best = {
                        'start': _start,
                        'end': _end,
                        'size': _size,
                        'pct': _pct
                    }
                break  #and go to the next _start

    #at the end, pull the best start and end combo...
    _startdt = _df['month_start'][_best['start']]
    _enddt = _df['month_start'][_best['end']]

    top_ranges.append(f"{_startdt:%b '%y} - {_enddt:%b '%y}")

artists['peak_range'] = top_ranges
```
</details>

In addition to the pure Top 10, I also wanted to try to tease out tracks and artists that have had *consistent* playtime throughout your life (or, your life on Spotify at least).  I suppose unsurprisingly in retrospect, there weren't that many hits for songs that have had consistent playtime over a user's **entire** time on Spotify, so I decided to just call out the top one for each.  For me, the results made a lot of sense and was cool to see how this highlighted a song that wasn't even in my Top 5:

![]({{ site.baseurl }}/assets/projects/20250811_spotifywrapped_siteassets/themesong_text.png)
<figcaption>My most consistently listened song, displayed in the application's Top Songs header text</figcaption>

<details>
  <summary>Full code and method (for nerds)</summary>
  The method here was to look for the songs and artists that had the highest median playtime across <strong>all</strong> months (including those with no playtime).  Could there be a better method for this?  Almost definitely.  But this was added pretty late in the project as a fun little callout and I felt like it satisfied the need.<br><br>

  Approach was just to blow out songs and artists into a full grid of each month and rank medians.  For obvious reasons, medians of zero are excluded:
  
  ``` python
  #what was your most CONSISTENT song?
  full_songs = pd.DataFrame({'song_name': streamhx['song_name'].unique()})
  full_grid = full_months.merge(full_songs, how="cross")
  
  song_months = streamhx.groupby(['song_name', 'month_start'], as_index=False).agg(ct=('ts', 'count'), hr_played=('hr_played', 'sum'))
  song_months = full_grid.merge(song_months, how="left", on=['month_start', 'song_name'])
  song_months = song_months.fillna(0)
  
  song_rank = song_months.groupby('song_name', as_index=False)['hr_played'].median()
  song_rank = song_rank.sort_values('hr_played', ascending=False)
  song_rank = song_rank[song_rank['hr_played']>0]  #ignore median 0
  
  goto_song = song_rank.head(1)['song_name'].iloc[0]
  goto_songartist = songs[songs['song_name']==goto_song].head(1)['artist_name'].iloc[0]
  ```
  
  Recognizing that zeros were being excluded and that may mean some users don't get any hits here - I coded this as an optional output in the app that only shows where applicable:
  
  ``` python
  ##Top Songs##
  _optionalphrase = ""
  if len(goto_song)>0: _optionalphrase = f"Your theme song for the past {datayears} years has been **{goto_song}** by **{goto_songartist}**.  While it may not have been your most played, this is the song you've listened to the most consistently through it all.  "
  
  st.header("Top Tracks")
  st.write(f"Out of the {streamhx['song_name'].nunique():,} unique songs you've listened to, a few were certainly your favourites.  "
           f"{_optionalphrase}"
           "Check out the full list of your top tracks below:"
           )
  st.dataframe(df_topsongs_display, height=387)
  ```
</details>


### Trending Artists
The second thing I wanted to do to take this a bit deeper was show the full historical listening relationship with each top artist.  The analysis was like it sounds, but it was very neat to see the ebbs and flows with each artist.  Specifically for me, it was super cool to see how my peak listening with each artist aligned with their major album releases:

![]({{ site.baseurl }}/assets/projects/20250811_spotifywrapped_siteassets/topartists_chart.png)
<figcaption>My all-time historical listening trends with each of my top artists</figcaption>

![]({{ site.baseurl }}/assets/projects/20250811_spotifywrapped_siteassets/1975_trendalbums.png)
![]({{ site.baseurl }}/assets/projects/20250811_spotifywrapped_siteassets/osooso_trendalbums.png)
<figcaption>Two really cool examples where you can see how my listening for The 1975 and Oso Oso peaked with each new album release</figcaption>

<details>
  <summary>Full code for nerds</summary>
  Pretty straightforward here - only real thing of note was transforming monthly playtime to a rolling 6mo avg. to smooth out spikes and highlight overall trends.  The in-app visual leverages plotly for an interactive visual, letting users filter or drill down to specific artists, and zoom into time ranges:

  ``` python
artist_month['hrs_perweek'] = artist_month['hr_played'] / 4.3  #avg weeks/month

artist_month['artist_name'] = pd.Categorical(artist_month['artist_name'], categories=artists.index, ordered=True)  #convert to categorical to allow sorting by top artists
artist_month = artist_month.sort_values(['artist_name', 'month_start'])  #sort (required upfront for accurate rolling)

artist_month['6mo_avg'] = (  #calculate a 6mo avg
    artist_month.groupby('artist_name', observed=False)['hrs_perweek'].transform(lambda x: x.rolling(window=6).mean())
)

fig = px.area(
    artist_month,
    x='month_start',
    y='6mo_avg',
    color='artist_name',
    line_shape='spline',
    labels = {
        'month_start': 'Month',
        'artist_name': 'Artist',
        '6mo_avg': 'Avg. Weekly Hours'
    },
    hover_data={'6mo_avg':':.1f'}  #set formatting for hover
)

fig.update_traces(line_width=0.5)
fig.update_layout(plot_bgcolor='white', xaxis_title='')

for f in [fig.update_xaxes, fig.update_yaxes]:  #iteratiely update both axis
    f(gridcolor='gainsboro', griddash='dot', gridwidth=0)
    ```
</details>


### Musical "Binges"
Another thing I wanted to do here was to call out those times we all get obsessed with one song for a week or two.  This is probably my favourite section of the whole analysis, as for me it was simultaneously a fun trip down memory lane, and a super embarassing exposé of guilty pleasures.  The high-level design here was to create a table for each song's plays per week and identify an "outlier" threshold for the music binges to highlight on the list:

![]({{ site.baseurl }}/assets/projects/20250811_spotifywrapped_siteassets/obsessions.png)
<figcaption>...Mortifying.  I can vividly remember each of these weeks lol</figcaption>

<details>
  <summary>Full code and method (for nerds)</summary>
  First up was generating the songs x weeks table.  This was also leveraged for the "top week" section of the Top Songs above:
  
  ``` python
  #biggest week
songweeks = streamhx.groupby(by=['song_name', 'artist_name', 'week_start'], as_index=False).agg(
    times_played=('ts','count'),
    total_hrs=('hr_played','sum')
)

_topids = songweeks.groupby(by=['song_name', 'artist_name'])['times_played'].idxmax()
best_weeks = songweeks.loc[_topids]
best_weeks.rename(columns={
    'week_start': 'top_week',
    'times_played': 'plays_top_week',
    'total_hrs': 'hrs_top_week'
}, inplace=True)
songs = songs.merge(best_weeks, on=['song_name', 'artist_name'])
songs.head(20)
```

In terms of creating the full list of "binges", the only real thing left to do here was identify a sufficiently embarassing threshold.  I tried to make this as dynamic as possible by referencing the 95th percentile of song weeks (as opposed to a hard cutoff on plays or hours, which may be very different for different people).<br><br>

I noticed one weird tweak in my data where some songs would have a ton of very short plays (i.e. lots of skips), so added a filter that at least 1 minute on avg. must hvae been played in the week.  I also limited to one top song per week, since I at least had a lot of weeks where every track on an album/playlist appeared and it crowded the list:

``` python
#what were your obsessions??
threshold = np.percentile(songweeks.groupby(by='week_start')['times_played'].max(), 95)  #make this dynamic to different listening styles
obsessions = songweeks[(songweeks['times_played']>=threshold) & (songweeks['total_hrs']>threshold*1/60)]  #there seem to be weird weeks with lots of very short plays... exclude

#just take the top 1 song per week
_topids = obsessions.groupby('week_start')['times_played'].idxmax()
obsessions = obsessions.loc[_topids]
obsessions.sort_values(by='week_start', inplace=True)
obsessions
```
</details>


## Overall Listening Patterns
Another cool thing we have access to with the Extended Streaming History is a full record of timestamps when users listen to music.  The first obvious thing we can do with this is see how much music we're listening to!  I also identified and highlighted here a "peak listening year" when users listened to the most music:

![]({{ site.baseurl }}/)
<figcaption>My all-time listening volume history.  At my peak I was listening to over 24 hrs/wk of music</figcaption>

<details>
  <summary>Full code for nerds</summary>
  The approach for peak hours here was the same as "peak listening range" for Top Artists.  I just rolled through each 12-month window in the user's play history and logged the window with the highest total playtime.  I also used plotly for interactive charting here and overlaid a rolling 6mo avg:
  
  ``` python
  weekly = streamhx.groupby(by='week_start', as_index=False)['hr_played'].sum()
weekly = weekly.sort_values('week_start')
weekly['6mo_avg'] = weekly['hr_played'].rolling(window=26).mean()

#find peak 12mo
best = {'start':0, 'end':0, 'hours':0}  #initialize
for _start in range(len(weekly)-51):
    _end = _start+51  #12mo window
    _hours = weekly.iloc[_start:_end]['hr_played'].sum()
    if _hours > best['hours']:
        best = {'start':_start, 'end':_end, 'hours':_hours}

best['startdt'] = weekly.iloc[best['start']]['week_start']
best['enddt'] = weekly.iloc[best['end']]['week_start'] + timedelta(days=6)  #end of week, not start

#last year
lastyr = weekly.iloc[-52:]['hr_played'].sum()
pct_change = (lastyr - best['hours']) / best['hours']

_peakyn = [True if best['start'] <= x <= best['end'] else False for x in weekly.index]
colors = ['Peak' if x else 'Weekly Hours' for x in _peakyn]

fig = px.bar(
    weekly,
    x='week_start',
    y='hr_played',
    color=colors,
    color_discrete_sequence = ['darkgrey', 'darkolivegreen'],
    labels = {
        'week_start': 'Week Starting',
        'color': 'Period',
        'hr_played': 'Weekly Hours'
    },
    hover_data={'hr_played':':.1f'}  #set formatting for hover    
)

fig.add_scatter(
    x=weekly['week_start'],
    y=weekly['6mo_avg'],
    mode='lines',
    line={'color':'dimgrey', 'dash':'dot', 'width':1.5},
    name='6mo Avg.'
)

fig.update_layout(plot_bgcolor='white', xaxis_title='')

for f in [fig.update_xaxes, fig.update_yaxes]:  #iteratiely update both axis
    f(gridcolor='gainsboro', griddash='dot', gridwidth=0)

fig.write_html("plotly_totalplaytime.html", include_plotlyjs="cdn", full_html=True)
fig.show()
```
</details>

The second cool thing we can do is create a heatmap for peak listening times!  In my case, I thought it was super cool that you can see me literally going to bed and waking up later on weekends, as well as that I apparently only tend to listen to music over lunch on Fridays:

![]({{ site.baseurl }}/)
<figcaption>Heatmap of my peak times listening to music</figcaption>

<details>
  <summary>Full code for nerds</summary>
  Pretty basic pivot and plot here.  I define a couple formatting patches to outline work hours and weekends (these get reused later):
  
  ``` python
  datadays = (timedata['dttm'].max() - timedata['dttm'].min()).days

timelabels = {}
for _ in range (24):  #quick lookup for time formatting
    _hr = (_%12) or 12
    timelabels[_] = f"{_hr}:00 {'AM' if _<12 else 'PM'}"

hmap = timedata.pivot_table(index='hour', columns='weekday', values='hr_played', aggfunc = lambda x: x.sum()/(datadays/7))
hmap.index = hmap.index.map(timelabels)  #am/pm time values
hmap = hmap[['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday' ,'Saturday', 'Sunday']]  #weekdays in order

plt.figure(figsize=(8,6))
ax = sns.heatmap(hmap, annot=False, fmt='.0%', cmap = 'coolwarm', annot_kws={'size':8}, vmin=0)

#define these patches so we can reuse them later
def wkday_patch():
    return patches.Rectangle(  #workday
        xy=(0,17),
        width=7,
        height=-8,
        fill=False,
        edgecolor='black',
        lw=1
    )

def wkend_patch():
    return patches.Rectangle(  #weekend
        xy=(5,24),
        width=2,
        height=-24,
        fill=False,
        edgecolor='black',
        lw=1
    )

def border_patch():
    return patches.Rectangle(  #full border
        xy=(0,24),
        width=7,
        height=-24,
        fill=False,
        edgecolor='black',
        lw=1
    )

ax.add_patch(wkday_patch())
ax.add_patch(wkend_patch())
ax.add_patch(border_patch())
  ```
</details>


## Genre Analysis
This is where it gets fun analytically, and was the biggest area where I wanted to add something substantially new.  For a massive dataset like this made up of so many different individual tracks and artists, I wanted a way to group them together into common "listening styles" so that a user could see the trends and evolution of their overall listening tastes.

Spotify's developer portal has a bunch of great APIs for querying genre and sentiment data on individual tracks and artists.  **Unfortunately**, I learned midway through that Spotify shut down free access to these APIs a few months ago, restricting them to internal use and partners... So we adapt!  While nowhere near as clean and standardized, Last.fm exposes their entire log of open sourced user tags through developer APIs.  I set up a protocol to grab these for a user's play history.

<details>
  <summary>Full code (for nerds who care about API calls and data cleansing)</summary>
  Last.fm's TopArtistTags API returns a dictionary of user tags attributed to artists, sorted and scored by relative usage.  To be kind on the API, I only called this info for artists making up the top 90% of playtime:

  ``` python
API_key = 'I'll keep this to myself :)'
url = 'http://ws.audioscrobbler.com/2.0/'

params = {
    'api_key': API_key,
    'method': 'artist.getTopTags',
    'format': 'json'
}

responses = {}
_c = 0  #tracking
for a in topartists:
    params['artist'] = a
    _resp = requests.get(url, params)
    responses[a] = _resp.json()

    if _c%50==0: print(f"{_c}/{len(topartists)}")
    _c+=1

responses = {artist: data for artist, data in responses.items() if 'toptags' in data}  #exclude responses without the toptags key (rare not found errors)
  ```

  I then blew these dictionaries out into a single long-format table, cleaned up tags/removed common non-insightful ones, and re-pivoted this back to wide format to create a single table with a row per artist and each tag as a "feature" column (containing it's relative user-attributed weight for the artist).<br><br>

  Along the way, I did need to cut the hundreds of different user tags down to a more meaningful list to create a reasonable feature set for clustering.  There were a lot of more dynamic ways I tried doing this (tags held by >x% of artists, cumulative total count threshold, etc.), but because the number of clusters ended up being fixed (more on this later), a fixed number of features here gave the most consistent results:

  ``` python
artist_tags_long = []                              
for artist, data in responses.items():  #blow out dictionaries to long list of features
    for tag in data['toptags']['tag'][:]:  #only take the top x tags? [:x]
        _tagname = tag['name']
        _ct = tag['count']
        artist_tags_long.append({'artist_name':artist, 'tag_name':_tagname, 'ct':_ct})  #only the top 3 tags

artist_tags_long = pd.DataFrame(artist_tags_long)  #convert to df
artist_tags_long['tag_name'] = artist_tags_long['tag_name'].str.title()  #convert to title case for better matching
artist_tags_long['tag_name'] = artist_tags_long['tag_name'].str.replace(f"[{string.punctuation}]", " ", regex=True)
artist_tags_long = artist_tags_long[~artist_tags_long['tag_name'].isin(['All', 'Canadian', 'Canada', 'Usa', 'American', "Male Vocalists", "Female Vocalists"])]  #get rid of that garbage "All" tag

#cut out noisy tags
tags_all = artist_tags_long.groupby('tag_name')['ct'].sum()  #cut down noisy tags
tags_keep = tags_all.sort_values(ascending=False).head(50).index  #top 100 only
artist_tags_long = artist_tags_long[artist_tags_long['tag_name'].isin(tags_keep)] 

artist_tags = artist_tags_long.pivot_table(index='artist_name', columns='tag_name', values='ct', aggfunc='max')  #convert to wide format  #for some reason, some artists have the same tag twice... use aggfunc=max
artist_tags = artist_tags.fillna(0)
  ```
</details>

Using these user-generated tags, I then fit a clustering model to identify the core musical styles that describe a user's dominant listening styles.  If you're a nerd like me, there's a doozy of a dropdown here on how exactly I got this model to give good classifications, but the "so what" is that it condenses the thousands of user-generated tags into a small handful of meaningful styles:

![]({{ site.baseurl }}/assets/projects/20250811_spotifywrapped_siteassets/styles_sample.png)
<figcaption>My top four listening styles, with the top artists making up each</figcaption>

<details>
  <summary>Full method for nerds like me (you were warned)</summary>
  If you clicked on this, I'm going to assume you were also nerdy enough to click on the last one so are familiar with the data structure we parsed out of the Last.fm tags above.  At its core, these tags are now just making up an n-dimensional dataset of artist features, so it's theoretically a simple clustering exercise.  The challenge was really just striking a good balance between accuracy and concision/usefulness of results.<br><br>

  I used KMeans clustering for this.  In a desperate moment of poorly fit results I experimented with DBScan and HDBScan (as well as PCA) to see if they gave meaningfully better results - but they didn't much so I opted for simplicity.  In addition to StandardScaling in the model pipeline, I also scaled values across <en>samples</en> (artists), since dropping tags disproportionately removed weighting from some artists, and I wanted the remaining dataset to represent an even "distribution" of artists' tags across the remaining features.  I used an elbow method to identify an optimum number of clusters:
  
  ``` python
#scale across ARTISTS, not features
scaled_tags = StandardScaler().fit_transform(artist_tags.T).T
artist_tags = pd.DataFrame(  #keep the index and column names
    scaled_tags,
    columns = artist_tags.columns,
    index = artist_tags.index
)


results = {}
_nrange = range(2,51)  #max 10-30 clusters
for n in _nrange:
    kmodel = Pipeline([
        ('scaler', StandardScaler()),
        #('pca', PCA(n_components=int(artist_tags.shape[1]/3))),
        ('kmeans', KMeans(n_clusters=n, random_state=69, n_init=100))
    ])
    kmodel.fit(artist_tags)  #don't need to drop artist_name because it's the index

    _inertia = kmodel.named_steps['kmeans'].inertia_
    _silhouette = silhouette_score(artist_tags, kmodel.named_steps['kmeans'].labels_)
    _DBI = davies_bouldin_score(artist_tags, kmodel.named_steps['kmeans'].labels_)

    results[n] = {
        'inertia':_inertia, 
        'silhouette':_silhouette, 
        'DBI':_DBI,
        'labels': kmodel.named_steps['kmeans'].labels_,
        'model': kmodel
    }
    
    if n%10==0: print(f"{n}/{max(_nrange)} complete")

#find the optimum n
dbis = pd.Series({key: val['DBI'] for key, val in results.items()})
dbi_roll = dbis.rolling(window=3).mean()
n = dbis.idxmin()
#n=20  #override  #spoiler - this is what goes in the final version

print(f"\nn={n}\n\nInertia: {results[n]['inertia']:.3f}\nDBI: {results[n]['DBI']:.3f}\n")

ax = pd.DataFrame.from_dict(results, orient='index').plot(y=['inertia', 'silhouette', 'DBI'], secondary_y=['inertia'])
ax.set_xlabel("n Clusters")
plt.show()
  ```

  ![]({{ site.baseurl }}/assets/projects/20250811_spotifywrapped_siteassets/kmeans_elbowresults.png)
  
  Originally, the plan was to implement a full elbow method on each user's results and carry on with an optimum cluster size.  This was swapped for a fixed "20 clusters" in the final version because: 1) There was a pretty small "acceptable" range of clusters that balanced meaningful groupings and readability of later results, and 2) The Streamlit server is much slower than my laptop and this took ages to calculate.<br><br>
  
  With the model fit, the last piece was just to name these clusters.  Below is a sample of the correlations my 20 clusters had with each tag:

  ``` python
clusters = pd.DataFrame(
    kmodel.named_steps['kmeans'].cluster_centers_,
    columns = artist_tags.columns
)

plt.figure(figsize=(14,10))
ax = sns.heatmap(clusters.T, annot=True, fmt='.2f', annot_kws={'size':5})
ax.set_xlabel("Cluster")
ax.set_ylabel("Tag")
plt.savefig("kmodel_fthmap.png")
  ```
  
  ![]({{ site.baseurl }}/assets/projects/20250811_spotifywrapped_siteassets/kmodel_fthmap.png)
  
  For the most part, there's at least one pretty dominant tag for each cluster.  I set up logic to name the clusters based on (up to) the top two most correlated tags.  I attempted to define a threshold (20% of maximum correlation) for a value tight enough to include in naming.  After that, I just mapped the cluster names back onto each artist in the original dataset:

  ``` python
naming_threshold = np.mean(clusters.max())/5  #could be different for different data
cnames = {}
for _c, r in clusters.iterrows():
    r = r.where(r>naming_threshold).dropna()  #only positive values
    _top2 = r.sort_values(ascending=False).index[:2]
    _name = '/'.join(_top2) if len(_top2)>0 else 'Other'

    cnames[_c] = _name

tagged_artists = pd.DataFrame({
    'artist_name': artist_tags.index,
    'cluster_num': kmodel.named_steps['kmeans'].labels_,
    'cluster_name': pd.Series(kmodel.named_steps['kmeans'].labels_).map(cnames)
})

tagged_artists['hr_played'] = tagged_artists['artist_name'].map(streamhx.groupby(by='artist_name')['hr_played'].sum())

for c, _ in tagged_artists.groupby('cluster_name')['hr_played'].sum().sort_values(ascending=False).items():
    display(tagged_artists[tagged_artists['cluster_name']==c].sort_values(by='hr_played', ascending=False).head(10))
  ```

  Finally, the last piece was to clean up clusters.  A few "catch-all" clusters appeared seemingly capturing a handful of artists that really didn't fit well anywhere else.  To keep the list clean, I limited the displayed clusters to just those explaining the top 90% of listening (so the final styles capture 81% of total listening for those keeping track).  This produces a final list of ~8-12 clean style clusters:

  ``` python
artist_labels = tagged_artists.set_index('artist_name')['cluster_name']
streamhx['cluster_name'] = streamhx['artist_name'].map(artist_labels)

_crank = streamhx.groupby('cluster_name')['hr_played'].sum().sort_values(ascending=False)
_cumpct = _crank.cumsum() / _crank.sum()

graph_clusters = _cumpct[_cumpct<=0.9].index  #only include categories that explain 90% of listening
graph_clusters = [c for c in graph_clusters if c != 'Other']  #don't bother showing that "other" either if it shows up
  ```
</details>



















