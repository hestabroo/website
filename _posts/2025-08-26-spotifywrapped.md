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
  
  {% highlight python %}
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
  {% endhighlight %}

  Besides that, I did some basic cleanup to filter out audiobooks and other lame not-music stuff, as well as to truncate a "tail" at the start of usage (this might have just been a me thing, but my account "existed" a year before I really started using it, so most charts had a year of whitespace).  Also added some QOL columns:
  
  {% highlight python %}
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
  {% endhighlight %}
  
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

  {% highlight python %}
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
{% endhighlight %}
</details>

In addition to the pure Top 10, I also wanted to try to tease out tracks and artists that have had *consistent* playtime throughout your life (or, your life on Spotify at least).  I suppose unsurprisingly in retrospect, there weren't that many hits for songs that have had consistent playtime over a user's **entire** time on Spotify, so I decided to just call out the top one for each.  For me, the results made a lot of sense and was cool to see how this highlighted a song that wasn't even in my Top 5:

![]({{ site.baseurl }}/assets/projects/20250811_spotifywrapped_siteassets/themesong_text.png)
<figcaption>My most consistently listened song, displayed in the application's Top Songs header text</figcaption>

<details>
  <summary>Full code for nerds</summary>
  The method here was to look for the songs and artists that had the highest median playtime across <strong>all</strong> months (including those with no playtime).  Could there be a better method for this?  Almost definitely.  But this was added pretty late in the project as a fun little callout and I felt like it satisfied the need.<br><br>

  Approach was just to blow out songs and artists into a full grid of each month and rank medians.  For obvious reasons, medians of zero are excluded:
  
  {% highlight python %}
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
  {% endhighlight %}
  
  Recognizing that zeros were being excluded and that may mean some users don't get any hits here - I coded this as an optional output in the app that only shows where applicable:
  
  {% highlight python %}
  ##Top Songs##
  _optionalphrase = ""
  if len(goto_song)>0: _optionalphrase = f"Your theme song for the past {datayears} years has been **{goto_song}** by **{goto_songartist}**.  While it may not have been your most played, this is the song you've listened to the most consistently through it all.  "
  
  st.header("Top Tracks")
  st.write(f"Out of the {streamhx['song_name'].nunique():,} unique songs you've listened to, a few were certainly your favourites.  "
           f"{_optionalphrase}"
           "Check out the full list of your top tracks below:"
           )
  st.dataframe(df_topsongs_display, height=387)
  {% endhighlight %}
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

  {% highlight python %}
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
    {% endhighlight %}
</details>


### Musical "Binges"
Another thing I wanted to do here was to call out those times we all get obsessed with one song for a week or two.  This is probably my favourite section of the whole analysis, as for me it was simultaneously a fun trip down memory lane, and a super embarassing exposé of guilty pleasures.  The high-level design here was to create a table for each song's plays per week and identify an "outlier" threshold for the music binges to highlight on the list:

![]({{ site.baseurl }}/assets/projects/20250811_spotifywrapped_siteassets/obsessions.png)
<figcaption>...Mortifying.  I can vividly remember each of these weeks lol</figcaption>

<details>
  <summary>Full code for nerds</summary>
  First up was generating the songs x weeks table.  This was also leveraged for the "top week" section of the Top Songs above:
  
  {% highlight python %}
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
{% endhighlight %}

In terms of creating the full list of "binges", the only real thing left to do here was identify a sufficiently embarassing threshold.  I tried to make this as dynamic as possible by referencing the 95th percentile of song weeks (as opposed to a hard cutoff on plays or hours, which may be very different for different people).<br><br>

I noticed one weird tweak in my data where some songs would have a ton of very short plays (i.e. lots of skips), so added a filter that at least 1 minute on avg. must hvae been played in the week.  I also limited to one top song per week, since I at least had a lot of weeks where every track on an album/playlist appeared and it crowded the list:

{% highlight python %}
#what were your obsessions??
threshold = np.percentile(songweeks.groupby(by='week_start')['times_played'].max(), 95)  #make this dynamic to different listening styles
obsessions = songweeks[(songweeks['times_played']>=threshold) & (songweeks['total_hrs']>threshold*1/60)]  #there seem to be weird weeks with lots of very short plays... exclude

#just take the top 1 song per week
_topids = obsessions.groupby('week_start')['times_played'].idxmax()
obsessions = obsessions.loc[_topids]
obsessions.sort_values(by='week_start', inplace=True)
obsessions
{% endhighlight %}
</details>


## Overall Listening Patterns











