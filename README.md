# MTA Ridership & Turnstile‑Hopping Data Visualization
### MTA Open Data Challenge – September 2024 → November 2024  

**Author:** Ibrahim Faruquee  

***

## Table of Contents
- [Project Overview](#project-overview)
- [Data Sources](#data-sources)
- [Technology Stack](#technology-stack)
- [Methodology](#methodology)
  - [4.1 Data Ingestion & Cleaning](#41-data-ingestion--cleaning)
  - [4.2 Ridership Trend Analysis & Animated Maps](#42-ridership-trend-analysis--animated-maps)
  - [4.3 Spatial Join – Turnstile‑Hopping ↔ NYPD Arrests](#43-spatial-join--turnstilehopping--nypd-arrests)
  - [4.4 Interactive Folium Map](#44-interactive-folium-map)
- [Key Findings & Insights](#key-findings--insights)
- [Deliverables](#deliverables)
- [Challenges & Lessons Learned](#challenges--lessons-learned)
- [Future Extensions](#future-extensions)
- [How to Re‑run the Notebook](#how-to-re-run-the-notebook)
- [References & Resources](#references--resources)

***

## Project Overview
The goal of this project was to combine two open‑data sets released by the Metropolitan Transportation Authority (MTA) and the New York Police Department (NYPD) to surface actionable insights for city planners, transit operators, and public‑safety stakeholders.

**Specific objectives:**
- Visualize spatio‑temporal subway ridership patterns across the entire NYC subway network for 2024.  
- Identify “turnstile‑hopping” incidents (theft‑of‑services arrests) and associate each incident with its nearest subway station.  
- Produce an enriched arrests table that includes the nearest station name and coordinates.  
- Deliver interactive visual assets (animated GIFs, Folium map) that allow stakeholders to explore the data dynamically.

***

## Data Sources
| Dataset | Description | URL (as of Sep 2024) |
|----------|--------------|---------------------|
| **MTA Subway Hourly Ridership** | Hourly boardings per station (latitude/longitude, timestamp, ridership count). | https://data.ny.gov/Transportation/MTA-Subway-Hourly-Ridership-Beginning-February-202… |
| **NYPD Arrests – Turnstile‑Jumping** | Historic arrest records where *PD description* contains “THEFT OF SERVICES.” Includes arrest date, location (lat/lon), and offense codes. | https://data.cityofnewyork.us/Public-Safety/NYPD-Arrests-Data-Historic-/8h9b‑rp9u/about_data |

Both files were downloaded as CSVs (`clean_subway.csv` and `NYPD_Arrests_small.csv`) and stored locally for reproducibility.

***

## Technology Stack
| Category | Libraries / Tools | Purpose |
|-----------|------------------|----------|
| Core Language | Python 3.11 | Main scripting environment |
| Data Manipulation | pandas, numpy | Loading, cleaning, aggregating |
| Statistical / Distance Ops | scipy.spatial.distance.cdist | Nearest‑station calculations |
| Visualization | matplotlib, seaborn, matplotlib.animation | Static & animated scatter plots |
| Basemap | contextily (CartoDB Positron, OSM) | Adding background tiles |
| Interactive Mapping | folium, folium.plugins.MarkerCluster | Stakeholder‑facing web map |
| Optional | tqdm, ipython.display | Progress bars & inline media |
| Export | pillow (GIF writer) | Saving animations as GIFs |

All dependencies can be installed via `pip` (see the notebook header).

***

## Methodology

### 4.1 Data Ingestion & Cleaning
```python
# Load CSVs
df_ridership = pd.read_csv('clean_subway.csv')
df_arrests   = pd.read_csv('NYPD_Arrests_small.csv')

# Drop exact duplicate rows
df_ridership.drop_duplicates(inplace=True)

# Convert timestamps to proper datetime objects
df_ridership['transit_timestamp'] = pd.to_datetime(df_ridership['transit_timestamp'])
```

**Key cleaning steps:**
- Removed duplicate records (≈ 0.1 % of ridership rows).  
- Ensured latitude/longitude columns are numeric and within NYC bounds.  
- Filtered arrests to only those whose *PD_DESC* contains “THEFT OF SERVICES” (case‑insensitive).

### 4.2 Ridership Trend Analysis & Animated Maps
**Hourly Aggregation:**
```python
df_ridership['hour'] = df_ridership['transit_timestamp'].dt.floor('H')
hourly_data = (
    df_ridership
    .groupby(['station_complex', 'hour', 'latitude', 'longitude'])['ridership']
    .mean()
    .reset_index()
)
```
*Averaging* was used to handle stations reporting multiple entries per hour.

**Normalization:**
```python
low, high = np.percentile(hourly_data['ridership'], [5, 95])
```

**Animated Scatterplots:**
- Full‑year GIF: `nyc_subway_hourly_ridership_in_2024_with_map.gif`  
- Average‑day 24‑frame GIF: `nyc_subway_average_hourly_ridership_with_map.gif`  
- Combined scatter + bar‑chart visualization: `nyc_subway_average_hourly_ridership_with_map_and_graph.gif`

### 4.3 Spatial Join – Turnstile‑Hopping ↔ NYPD Arrests
**Nearest‑Station Function:**
```python
def find_nearest_station(lat, lon, stations):
    arrest_coords = np.array([[lat, lon]])
    station_coords = stations[['latitude', 'longitude']].values
    distances = cdist(arrest_coords, station_coords)
    nearest_idx = distances.argmin()
    return stations.iloc[nearest_idx]['station_complex']
```

Each arrest was mapped to the nearest subway station using Euclidean distance.

**Join Application & Aggregation:**
- Enriched arrests CSV: `enriched_turnstile_jumping_arrests.csv`  
- Summary table prepared for mapping (`result_df`).

### 4.4 Interactive Folium Map
An interactive map visualizes turnstile‑hopping concentrations city‑wide.

Key features:
- Marker clustering for readability.  
- Circle marker size proportional to incident count.  
- Pop‑ups show exact numbers and station names.  

Output: `turnstile_hoppers_map_oct_to_dec_2023.html`

***

## Key Findings & Insights
| Insight | Evidence |
|----------|-----------|
| Peak ridership hours | Daily‑average map shows spikes at 07:00‑09:00 and 17:00‑19:00 in Manhattan, Brooklyn, and Queens. |
| Geographic hotspots | Times Sq‑42 St, Grand Central‑42 St, 34 St‑Penn Station, and Atlantic Ave‑Barclays Center. |
| Turnstile‑hopping concentration | Top three: 3 Av‑149 St (12 incidents), Times Sq‑42 St (8), Tremont Av (7). |
| Temporal mismatch | Arrests (Oct‑Dec 2023) do not align with 2024 ridership peaks. |
| Spatial correlation | Jumpings occur near busy transfer hubs. |
| Data completeness | Arrest dataset ends Dec 2023; extending data would allow full‑year comparison. |

***

## Deliverables
| Asset | Format | Description |
|--------|---------|-------------|
| Animated yearly ridership GIF | `nyc_subway_hourly_ridership_in_2024_with_map.gif` | Hour‑by‑hour scatter over 2024. |
| Average‑day ridership GIF | `nyc_subway_average_hourly_ridership_with_map.gif` | 24‑frame loop of daily ridership. |
| Combined scatter + bar‑chart GIF | `nyc_subway_average_hourly_ridership_with_map_and_graph.gif` | Spatial + top‑10 busiest stations per hour. |
| Enriched arrests CSV | `enriched_turnstile_jumping_arrests.csv` | Arrests with nearest station names. |
| Turnstile‑hopping summary CSV | `result_df` | Station, count, lat/lon for mapping. |
| Interactive Folium map | `turnstile_hoppers_map_oct_to_dec_2023.html` | Clickable, clustered map. |
| Jupyter Notebook | `MTA_Ridership_Turnstile_Hopping.ipynb` | Full reproducible codebase. |
| Project README | `README.md` | Overview, methods, and future directions. |

***

## Challenges & Lessons Learned
- **Coordinate Precision vs. Metric** – Euclidean distance suffices locally, but haversine is more accurate for broader ranges.  
- **Missing Station IDs** – Required custom nearest‑neighbor logic; GIS solutions (e.g., `geopandas.sjoin_nearest`) could improve performance.  
- **Large Animation Files** – Yearly GIF (~300 MB) is unwieldy; MP4 export via *ffmpeg* preferred.  
- **Temporal Coverage Gap** – Arrest data limited to Q4 2023; full‑year data would strengthen insights.

***

## Future Extensions
| Idea | Benefit |
|-------|----------|
| Haversine‑based matching | More accurate spatial joins near borough borders. |
| Heat‑map rasterization | Smooth density visualization. |
| Machine‑learning forecasting | Predict ridership spikes for proactive planning. |
| Integrate additional crime types | Examine broader safety patterns. |
| Interactive dashboard | Real‑time stakeholder access via Flask/Dash. |
| Public API wrapper | Reusable ingestion and processing pipeline. |

***

## How to Re‑run the Notebook
1. Clone or download the repository and navigate to the directory.  
2. Create a virtual environment:
   ```bash
   python -m venv venv
   source venv/bin/activate   # Windows: venv\Scripts\activate
   pip install -r requirements.txt
   ```
3. `requirements.txt` should include:
   ```
   pandas
   numpy
   matplotlib
   seaborn
   scipy
   folium
   contextily
   tqdm
   ipython
   pillow
   ```
4. Place `clean_subway.csv` and `NYPD_Arrests_small.csv` in the notebook directory.  
5. Open and run the notebook sequentially in JupyterLab or Jupyter Notebook.  
6. Trust the notebook before rendering Folium maps.  
7. Resulting outputs (GIFs, HTML map) will appear in the working directory.

***

## References & Resources
- **MTA Open Data Portal:** https://data.ny.gov/Transportation/MTA-Subway-Hourly-Ridership-Beginning-February-202…  
- **NYPD Arrest Data:** https://data.cityofnewyork.us/Public-Safety/NYPD-Arrests-Data-Historic-/8h9b‑rp9u/about_data  
- **Contextily Basemaps:** [https://github.com/darribas/contextily](https://github.com/darribas/contextily)  
- **Folium Documentation:** [https://python-visualization.github.io/folium/](https://python-visualization.github.io/folium/)  
- **Matplotlib Animation Tutorial:** [https://matplotlib.org/stable/api/_as_gen/matplotlib.animation.FuncAnimation.html](https://matplotlib.org/stable/api/_as_gen/matplotlib.animation.FuncAnimation.html)  

Prepared for the **MTA Open Data Challenge** submission.
