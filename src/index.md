# CSC477 Assignment 5: Interactive Visualization 
**By Megan Fung**

---

## Spotify Danceability vs Popularity

**Do more danceable songs tend to be more popular?**

This interactive visualization observes the relationship between a track's **danceability** and its **popularity** across more than one hundred Spotify genres. By combining musical attributes with popularity metrics, this visualization explores how different genres align or diverge from mainstream trends. 

---

## Dataset Overview

The **[Spotify Tracks Dataset-- Maharshi Pandya, Kaggle](https://www.kaggle.com/datasets/maharshipandya/-spotify-tracks-dataset/data)**, sourced from Kaggle, is a community complied dataset that aggregates track level information from Spotify's public API. The dataset contains: 
- tracks (one row per track) across 125 different genres
- includes metadata (track name, artist, album) and audio features (danceability, energy, valence, etc.)
- and popularity level, which represents how much the track is being played on Spotify on a 0-100 scale 

| Field | Description |
|--------|--------------|
| **track_id** | Unique id for each Spotify track |
| **track_name** | Name of the track |
| **artists** | Performing artist(s) |
| **track_genre** | Primary genre classification |
| **popularity** | Spotify popularity score (0–100) based on recent streams |
| **danceability** | Suitability of a track for dancing (0–1) |
| **valence** | Musical positivity (0–1) |
| **duration_ms** | Track length in milliseconds |
| **explicit** | Boolean flag indicating explicit lyrics |

*This dataset was last updated in 2022.*

---

## Visual Encodings + Features

| Axis | Description |
|--------|--------------|
| **X** | **Danceability**: how suitable the track is for dancing |
| **Y** | **Popularity**: how popular the track is on Spotify (based on play counts) |

The visualization offers interactive features for exploratory analysis: 
- **genre** dropdown menu: select a specific genre or view all genres
   - switch between genres (eg. pop, edm, jazz) to see how danceability relates to popularity 
- **popularity** slider: adjust minimum popularity threshold (0-100 scale)
   - focus on mainstream hits (> 60) vs all tracks to see how correlations strengthens/ weakens
- **track level details** hover tooltip: view track specific metadata 
- **quadrant distribution**: reveals distribution of tracks that are danceable but 'underground' vs those that achieve broad mainstream appeal 
- **top 5** most popular tracks: displays top tracks based on filters 

Based on user selection, visuals and statistics update in real time. 

--- 

```js
// load data
const raw_data = await FileAttachment("data/spotify-tracks-dataset.csv").csv();
```

```js
// parse and clean data
const cleaned_data = raw_data
  .map(d => ({
    track_id: d.track_id?.trim(),
    track_name: d.track_name?.trim(),
    artists: d.artists?.trim(),
    track_genre: d.track_genre?.trim() || 'Unknown',
    popularity: +d.popularity,
    danceability: +d.danceability,
    valence: +d.valence,
    duration_ms: +d.duration_ms,
    explicit: d.explicit === 'True' || d.explicit === 'true'
  }))
  // invalid/ incomplete records
  .filter(d => 
    // must have all field values for analysis
    d.track_id &&
    d.track_name && 
    d.track_name.length > 0 &&
    d.artists && 
    d.artists.length > 0 &&
    d.track_genre !== 'Unknown' &&
    
    // sanity check: numerical values
    !isNaN(d.popularity) && 
    !isNaN(d.danceability) && 
    !isNaN(d.valence) &&
    !isNaN(d.duration_ms) &&
    d.danceability >= 0 && d.danceability <= 1 &&
    d.valence >= 0 && d.valence <= 1 &&
    d.popularity >= 0 && d.popularity <= 100 &&
    
    // sanity check:  duration values 
    d.duration_ms >= 30000 && d.duration_ms <= 1800000 &&
    
    // sanity check: invalid records (test/ untitled records)
    !d.track_name.toLowerCase().includes('test') &&
    !d.track_name.toLowerCase().includes('untitled') &&
    d.artists !== 'Unknown' &&
    d.artists !== 'Various Artists'
  );

// remove duplicate records
const seen_tracks = new Set();
const spotify_data = cleaned_data.filter(d => {
  if (seen_tracks.has(d.track_id)) {
    return false;
  }
  seen_tracks.add(d.track_id);
  return true;
});
```

## Interactive Controls

```js
import * as Plot from "@observablehq/plot";
```

```js
// genre dropdown
const genres = ['All', ...Array.from(new Set(spotify_data.map(d => d.track_genre))).sort()];
const selectedGenre = view(Inputs.select(genres, {
  label: "Select Genre:",
  value: "All"
}));
```

```js
// popularity slider (set minimum threshold)
const minPopularity = view(Inputs.range([0, 100], {
  label: "Minimum Popularity:",
  value: 0,
  step: 1
}));
```

```js
// filter data based on user selections
const filtered_data = spotify_data.filter(d => 
  (selectedGenre === 'All' || d.track_genre === selectedGenre) &&
  d.popularity >= minPopularity
);
```

Showing **${filtered_data.length.toLocaleString()}** tracks

---

## Visualization & Analysis

<div style="display: grid; grid-template-columns: 1fr 400px; gap: 30px; margin: 20px 0;">

<div style="min-width: 0;">

### Bubble Chart

```html
<style>
  .bubble-chart-container svg text[aria-label*="axis label"],
  .bubble-chart-container svg text[text-anchor="middle"][font-size="10"],
  .bubble-chart-container svg text[text-anchor="end"][font-size="10"] {
    font-size: 18px !important;
    font-weight: 900 !important;
  }
</style>
```

```js
html`<div class="bubble-chart-container">${
  Plot.plot({
    width: 900,
    height: 600,
    marginLeft: 90,
    marginBottom: 80,
    marginRight: 20,
    marginTop: 20,
    
    marks: [
      // background frame
      Plot.frame({stroke: "#e0e0e0"}),
      
      // quadrant lines
      Plot.ruleX([0.5], {stroke: "#ddd", strokeDasharray: "4 4"}),
      Plot.ruleY([50], {stroke: "#ddd", strokeDasharray: "4 4"}),
      
      // bubble chart 
      Plot.dot(filtered_data, {
        x: "danceability",
        y: "popularity",
        r: 3.5,  // fixed bubble size (change to use valence if it adds valueable information to narrative)
        fill: "track_genre",
        fillOpacity: selectedGenre === 'All' ? 0.5 : 0.7,  // transparency based off genre selection 
        stroke: "white",
        strokeWidth: 0.5,
        tip: true,
        title: d => `Track: ${d.track_name}\nArtist: ${d.artists}\nGenre: ${d.track_genre}\nPopularity: ${d.popularity}\nDanceability: ${d.danceability.toFixed(3)}\nValence (Happiness): ${d.valence.toFixed(3)}\nDuration: ${Math.floor(d.duration_ms / 60000)}:${String(Math.floor((d.duration_ms % 60000) / 1000)).padStart(2, '0')}\nExplicit: ${d.explicit ? 'Yes' : 'No'}`
      })
    ],
    
    x: {
      label: "Danceability (0 = not danceable, 1 = very danceable) →",
      domain: [0, 1],
      grid: true,
      tickSize: 6
    },
    
    y: {
      label: "Popularity (0-100) →",
      domain: [0, 100],
      grid: true,
      labelAnchor: "center",
      tickSize: 6
    },
    
    color: {
      legend: selectedGenre === 'All',  
      label: "Genre",
      scheme: "tableau10",
      domain: selectedGenre === 'All' ? undefined : [selectedGenre]
    },
    
    style: {
      fontSize: "16px",
      background: "white"
    }
  })
}</div>`
```

</div>

<div style="min-width: 0;">

```js
// statistics for current filtered view
const stats = {
  count: filtered_data.length,
  avgPopularity: d3.mean(filtered_data, d => d.popularity) || 0,
  avgDanceability: d3.mean(filtered_data, d => d.danceability) || 0,
  avgValence: d3.mean(filtered_data, d => d.valence) || 0,
  medianPopularity: d3.median(filtered_data, d => d.popularity) || 0,
  medianDanceability: d3.median(filtered_data, d => d.danceability) || 0,
  explicitCount: filtered_data.filter(d => d.explicit).length
};

// quadrant distribution - where do tracks cluster?
const quadrants = {
  highDanceHighPop: filtered_data.filter(d => d.danceability >= 0.5 && d.popularity >= 50).length,
  highDanceLowPop: filtered_data.filter(d => d.danceability >= 0.5 && d.popularity < 50).length,
  lowDanceHighPop: filtered_data.filter(d => d.danceability < 0.5 && d.popularity >= 50).length,
  lowDanceLowPop: filtered_data.filter(d => d.danceability < 0.5 && d.popularity < 50).length
};

// top 5 most popular tracks in current selection
const topTracks = filtered_data
  .sort((a, b) => b.popularity - a.popularity)
  .slice(0, 5);
```

### Top 5 Most Popular Tracks

```html
<style>
  .track-tooltip {
    position: relative;
    cursor: help;
    border-bottom: 1px dotted #666;
    display: inline-block;
  }
  
  .track-tooltip:hover::after {
    content: attr(data-tooltip);
    position: absolute;
    left: 0;
    top: 100%;
    margin-top: 8px;
    padding: 12px;
    background: #333;
    color: white;
    border-radius: 6px;
    font-size: 13px;
    white-space: pre-line;
    z-index: 1000;
    min-width: 250px;
    box-shadow: 0 4px 12px rgba(0,0,0,0.3);
    pointer-events: none;
  }
  
  .track-tooltip:hover::before {
    content: '';
    position: absolute;
    left: 10px;
    top: 100%;
    margin-top: 2px;
    border: 6px solid transparent;
    border-bottom-color: #333;
    z-index: 1001;
    pointer-events: none;
  }
</style>
```

```js
html`<div style="background: white; padding: 15px; border: 1px solid #ddd; border-radius: 8px; margin-bottom: 20px;">
  <table style="width: 100%; border-collapse: collapse;">
    <thead>
      <tr style="border-bottom: 2px solid #1db954;">
        <th style="padding: 12px; text-align: left; font-weight: bold; color: #333; font-size: 14px;">Track</th>
        <th style="padding: 12px; text-align: left; font-weight: bold; color: #333; font-size: 14px;">Genre</th>
      </tr>
    </thead>
    <tbody>
      ${topTracks.map((track, i) => {
        const duration = `${Math.floor(track.duration_ms / 60000)}:${String(Math.floor((track.duration_ms % 60000) / 1000)).padStart(2, '0')}`;
        const tooltipText = `Artist: ${track.artists}\nPopularity: ${track.popularity}\nDanceability: ${track.danceability.toFixed(3)}\nValence: ${track.valence.toFixed(3)}\nDuration: ${duration}\nExplicit: ${track.explicit ? 'Yes' : 'No'}`;
        return html`<tr style="border-bottom: 1px solid #eee; background: ${i % 2 === 0 ? '#f9f9f9' : 'white'};">
          <td style="padding: 12px; vertical-align: middle;">
            <span class="track-tooltip" data-tooltip=${tooltipText}>${track.track_name}</span>
          </td>
          <td style="padding: 12px; color: #666; vertical-align: middle;">${track.track_genre}</td>
        </tr>`;
      })}
    </tbody>
  </table>
</div>`
```

### Quadrant Distribution

```js
html`<div style="background: white; padding: 15px; border: 1px solid #ddd; border-radius: 8px;">
  <p style="margin: 0 0 15px 0; font-size: 0.9em; color: #666;">Where do tracks cluster?</p>
  <div style="display: grid; grid-template-columns: 1fr 1fr; gap: 10px;">
    <div style="text-align: center; padding: 12px; background: #e8f5e9; border-radius: 6px;">
      <div style="font-size: 1.6em; font-weight: bold; color: #2e7d32;">${quadrants.highDanceHighPop}</div>
      <div style="font-size: 0.75em; color: #666;">High Both</div>
      <div style="font-size: 0.7em; color: #888;">${((quadrants.highDanceHighPop / stats.count) * 100).toFixed(1)}%</div>
    </div>
    <div style="text-align: center; padding: 12px; background: #fff3e0; border-radius: 6px;">
      <div style="font-size: 1.6em; font-weight: bold; color: #e65100;">${quadrants.lowDanceHighPop}</div>
      <div style="font-size: 0.75em; color: #666;">Low D, High P</div>
      <div style="font-size: 0.7em; color: #888;">${((quadrants.lowDanceHighPop / stats.count) * 100).toFixed(1)}%</div>
    </div>
    <div style="text-align: center; padding: 12px; background: #f3e5f5; border-radius: 6px;">
      <div style="font-size: 1.6em; font-weight: bold; color: #6a1b9a;">${quadrants.highDanceLowPop}</div>
      <div style="font-size: 0.75em; color: #666;">High D, Low P</div>
      <div style="font-size: 0.7em; color: #888;">${((quadrants.highDanceLowPop / stats.count) * 100).toFixed(1)}%</div>
    </div>
    <div style="text-align: center; padding: 12px; background: #eceff1; border-radius: 6px;">
      <div style="font-size: 1.6em; font-weight: bold; color: #455a64;">${quadrants.lowDanceLowPop}</div>
      <div style="font-size: 0.75em; color: #666;">Low Both</div>
      <div style="font-size: 0.7em; color: #888;">${((quadrants.lowDanceLowPop / stats.count) * 100).toFixed(1)}%</div>
    </div>
  </div>
</div>`
```

</div>

</div>

### Summary Statistics

```js
html`<div style="background: #f5f5f5; padding: 30px; border-radius: 8px; margin: 30px 0;">
  <h4 style="margin-top: 0; color: #1db954; text-align: left;">Current Selection: ${selectedGenre}</h4>
  <div style="display: grid; grid-template-columns: repeat(5, 1fr); gap: 20px; margin-top: 20px;">
    <div style="text-align: center;">
      <div style="font-size: 2.5em; font-weight: bold; color: #1db954;">${stats.count.toLocaleString()}</div>
      <div style="color: #666; font-size: 1em; margin-top: 5px;">Total Tracks</div>
    </div>
    <div style="text-align: center;">
      <div style="font-size: 2.5em; font-weight: bold; color: #1db954;">${(stats.avgPopularity || 0).toFixed(1)}</div>
      <div style="color: #666; font-size: 1em; margin-top: 5px;">Avg Popularity</div>
    </div>
    <div style="text-align: center;">
      <div style="font-size: 2.5em; font-weight: bold; color: #1db954;">${(stats.avgDanceability || 0).toFixed(3)}</div>
      <div style="color: #666; font-size: 1em; margin-top: 5px;">Avg Danceability</div>
    </div>
    <div style="text-align: center;">
      <div style="font-size: 2.5em; font-weight: bold; color: #1db954;">${(stats.avgValence || 0).toFixed(3)}</div>
      <div style="color: #666; font-size: 1em; margin-top: 5px;">Avg Valence</div>
    </div>
    <div style="text-align: center;">
      <div style="font-size: 2.5em; font-weight: bold; color: #1db954;">${((stats.explicitCount / stats.count) * 100).toFixed(1)}%</div>
      <div style="color: #666; font-size: 1em; margin-top: 5px;">Explicit Content</div>
    </div>
  </div>
</div>`
```

---

## Write-Up

For this assignment, I developed an interactive visualization that investigates whether more danceable songs tend to be more popular across Spotify genres. The visualization uses data from the **[Spotify Tracks Dataset](https://www.kaggle.com/datasets/maharshipandya/-spotify-tracks-dataset/data)**, a community-compiled resource built from Spotify's public API that provides track-level metadata and quantitative audio features such as danceability, valence, and popularity scores. This visualization aims to provide a data-driven interface that allows users to explore how musical characteristics align with or diverge from mainstream listening trends. 

After brainstorming different ways to visualize the dataset, I chose to implement a bubble chart scatter plot because it best reveals correlations between two continuous variables. Each bubble in the chart represents an individual track, plotted by danceability on the x-axis and popularity on the y-axis, with color encoding the track's genre. While I originally planned to scale bubble size by valence to convey mood variation, I found that incorporating this additional visual encoding was distracting and visually dense at scale. Therefore, I fixed the bubble size to create a more uniform field of points that emphasizes the relationship between the two core dimensions—**danceability** and **popularity**—without unnecessary visual noise. 

Designing the chart's interactivity transformed my static bubble chart into a more exploratory analytical tool. The chart takes input from two interactive controls: (1) a **genre dropdown** and (2) a **popularity slider**. The genre dropdown allows users to isolate tracks by genre, while the popularity slider allows users to set a minimum threshold for popularity scores to examine mainstream hits, 'underground' tracks, or the full dataset. These two filters enable users to dynamically filter the data and see, in real time, how the visual distribution and accompanying statistics change. These reactive features encourage users to conduct focused comparisons, such as contrasting Pop against Jazz or EDM against Classical, without overwhelming the display.

Along with the main bubble chart, I implemented additional supporting components to complement the visualization. To the right, a **Top 5 Most Popular Tracks** table dynamically updates with the data rendered in the main bubble chart. Both this table and the chart feature hover tooltips that present track-level metadata—artist, genre, valence, duration, and explicit flag—to provide deeper context. Below this table, a **Quadrant Distribution** panel summarizes how tracks cluster within four behavioral regions defined by median thresholds of danceability and popularity: high danceability/high popularity, low danceability/high popularity, high danceability/low popularity, and low danceability/low popularity. These values allow users to understand the proportion of tracks in each danceability/popularity pairing, highlighting correlations between our two narrative variables. 

Below the bubble chart, **Summary Statistics** display average values for the quantitative data being visualized. Together, the bubble chart and additional metrics form a coordinated multi-view design that connects micro-level exploration (individual songs) with macro-level trends (genre patterns). 

The visualization's layout follows a two-column structure to balance analytical detail with readability. The bubble chart, as the main visualization, occupies the left column as the focal view, while the right column contains contextual panels that serve as dynamically updated callouts users can examine while observing the main chart. The summary statistics below highlight key quantitative descriptors—total number of tracks, average popularity, average danceability, average valence, and proportion of explicit content. This visual structure allows users to interpret the data through the bubble chart while the additional components (side panels and summary statistics) direct focus toward the central relationship between danceability and popularity. 

The development process, from design to implementation, took approximately twelve hours total. My workflow consisted of five stages: exploration, sketching, implementation, refinement, and testing. 

During **exploration**, I examined the dataset for potential feature pairings. After careful analysis, I found that danceability and popularity formed the most compelling combination, as they offered clear numerical scales and intuitive interpretability. I then moved to **sketching**, where I brainstormed different ways to visualize this narrative. After creating a bar chart, a line chart, and a bubble chart, I found that the bubble chart best communicated the narrative at an individual track level while maintaining the ability to examine trends across clusters of tracks in specific genres. 

In the **implementation** phase, I built my sketch using Observable Plot. I first applied data cleaning logic—validating field values, filtering incomplete or unrealistic records, and removing duplicates—an essential preprocessing step to prevent misleading patterns and ensure accurate results. I then implemented each layer of interactivity individually to observe how it affected the bubble chart. I found that these two filters offered the most straightforward entry points for data exploration. Next, I implemented custom tooltips so that tracks could be individually examined using track-level data, allowing users to explore how patterns form without overwhelming the main display. 

During **refinement**, I implemented the additional summary panels to pull all the data together and strengthen my narrative. This step elevated the data visualized in the bubble chart by directing user focus to danceability and popularity scores and the overall composition of patterns. This phase consumed the most time, as determining the best layout for displaying all this information cohesively required considerable experimentation. Finally, during **testing**, I compared different filter combinations to optimize the patterns and trends revealed by the visualization. 

Overall, this project deepened my understanding of how interaction design shapes data interpretation. I learned that interactivity is most powerful when it is purposeful and minimal. Implementing controls that have a clear analytical function is essential to allow meaningful patterns to emerge from the data visualization. Through the tool I implemented, I discovered that genres such as Pop and EDM exhibit a strong positive relationship between danceability and popularity, confirming that high danceability tends to align with mainstream success. In contrast, genres with a wider range of danceability, like Classical music, exhibit weak correlations with popularity, suggesting that other musical qualities—such as emotional tone or instrumentation—drive listener engagement. The overall pattern shows that high danceability typically corresponds with high popularity, as revealed in the **Quadrant Distribution** panel.

In conclusion, Spotify Danceability vs Popularity demonstrates how focused design and interactive features can yield an engaging and insightful exploration experience. The visualization balances simplicity with depth, allowing users to examine the overall relationship between danceability and popularity while retaining the ability to isolate specific examples. Through the development process of this assignment, I gained a deeper understanding of how thoughtful design choices can bridge data analytics and curiosity.

---

## Source Code 
**[GitHub Repo](https://github.com/meganfung38/CSC477_Asgn5.git)**

