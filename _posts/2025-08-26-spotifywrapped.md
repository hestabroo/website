---
layout: single
title: "Spotify Wrapped+"
post_header: true
subtitle: "Creating an easy, interactive tool to analyze *all-time* Spotify listening trends"
excerpt: "Creating an easy, interactive tool to analyze *all-time* Spotify listening trends"
author_profile: false
tags: [Machine Learning, Dashboards]
image: "/assets/project_assets/20250826_spotifywrapped/spotifywraped_app_screenshot.png"
date: 2025-08-26
---
![Landing page for the "Spotify Wrapped+" app](/assets/projects/20250811_spotifywrapped_siteassets/spotifywrapped_app_screenshot.png)

*tl;dr Check out [this neat tool](https://haydenestabrook-spotifywrapped.streamlit.app/) I built to analyze your all-time Spotify listening history!*

## Intro
Music is a huge part of my life, and as someone obsessed with data it should surprise no one that I love Spotify Wrapped season every December.  I love the opportunity to see what my friends have been listening to, discover new artists, and enjoy the embarassing outings of everyone's guilty pleasure songs. 

*However*, I've always felt like the Wrapped summary you get each year is quite surface level, and it felt like a miss given the wealth of data Spotify has that they don't really go beyond "top 10 tracks/artists".  SO, I set out to make a "better" Spotify Wrapped that can go deeper into your personal listening style and look more broadly outside the vacuum of the past 12 months.

## Getting The Data
Spotify has (*had*...spoilers) some great developer APIs to query user play history and track info.  However, the API play history dataset is limited to just the past 50 songs and some basic top tracks/artists info.  

I wanted to build a tool that gave a more comprehensive picture of your listening, so I built the tool around Spotify's "Extended Streaming History", which is an all-time data dump you can request from Spotify.  Unfortunately, this did remove the ability to have an instant sign-on/OAuth UI for the app - but I felt like the pros outweighed the cons here.

The Extended Streaming History comes as a zipped json package, and was honestly pretty plug and play.  Some quick minor data cleanup and helper columns and we were good to go!

{% raw %}
<details>
  <summary>Full code for nerds</summary>
  
  The app is hosted on Streamlit.  Very simple out-of-the-box file uploader, and set up basic unpacking of the zip:
  
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

  Besides that, I did some basic cleanup to filter out audiobooks and other lame not-music stuff, as well as to truncate a "tail" at the start of usage (this might have just been a me thing, but my account "existed" ~a year before I really started using it, so most charts had a year of whitespace).  Also added some QOL columns:
  
  <pre><code class="language-python">
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
  </code></pre>
  
</details>{% endraw %}

## The Analysis
First up was the basics.  Critical as I was of Spotify only really doing the "top 10s", obviosuly it's something users will want to see, and a pretty cool thing to see across *all time*.  I tried to add a little more info around how many cumulative hours have been spent listening to each track/artist, as well as the "peak listening periods" for each:

![My top artists...](/assets/projects/20250811_spotifywrapped_siteassets/topartists_table.png)

![...and top songs.  For the uninitated, The Dirty Nelsons is the band I drum in... embarassing.](/assets/projects/20250811_spotifywrapped_siteassets/topsongs_table.png)

<details>
  <summary>Full code for nerds</summary>
  Don't need to bore you with the basics of finding top artists/tracks.  Only interesting thing here was the identification of a "peak listening window".  A bit arbitrarily, I defined this to be the smallest possible window containing at least 50% of the artist's playtime.  
  
  At first I tried to do this by iteratively "shrinking" the full date, dropping the lowest-volume end - but this greedy logic got hung up on local peaks.  I ended up looping through each possible window start point and extending *outwards* until 50% was captured, and finding the best (smallest) window that achieved this.
</details>











