<div align="center">

# trialx


</div>


## Quick Start

### Prerequisites

- Python 3.11 (other verisons might not work)
- A free [Copernicus Data Space](https://dataspace.copernicus.eu/) account (takes 2 minutes)
- The trained RF model file (in the trialx sub directory)
- Be patient it takes time soemtimes pinging mirrors for the road data

### Install and run

```bash
# Clone the repository
git https://github.com/NovaAarchi5120/tests-
cd Trial-
 
# Create and activate a virtual environment
python3.11 -m venv venv
 
# On macOS / Linux:
source venv/bin/activate
 
# On Windows:
venv\Scripts\activate

cd trialx
 
# Install dependencies
pip install -r requirements.txt
 
# Start the server
python trialx.py
```
 
The server starts on port 8000. Open your browser and go to:
 
```
http://localhost:8000
```

Open `http://localhost:8000`. On first launch a connect window opens automatically — enter your Client ID and Client Secret to link the satellite API (you can also open it anytime via **Connect** on the **Copernicus Link** card in the sidebar). That is it. No config files needed.

If you prefer environment variables instead of the UI, you can copy `.env.example` to `.env` and fill in your keys there. Both methods work. The connect window is just easier for most people.

## Using the Interface

trialx is designed to be used directly from the browser with minimal setup.

### Selecting an Area of Interest (AOI)

- Right-click anywhere on the map  
- Click **“Draw AOI”**  
- Drag to define your analysis region  

The system will automatically:
- detect road corridors in the selected area  
- fetch satellite data  
- begin analysis  

---

### Recommended Settings

For best results:

- **Mode:** Use **Dense Mode** for maximum data coverage  
- **Timeline:** Adjust based on your use case:
  - Short (1–2 weeks) → quick signals  
  - Medium (1–3 months) → trend detection  
  - Long (3–6 months) → strong baseline + anomaly detection  

Dense mode increases the number of images processed, improving detection reliability, especially in regions with cloud cover.



### Data Storage

By default, all cached data and detection outputs go to `trialx_data/` in the project directory. To redirect (for example, to an external drive with more space):

```bash
export trialx_DATA_DIR=/path/to/your/storage
```

### Environment Variables

<img width="1702" height="1073" alt="Image" src="https://github.com/user-attachments/assets/4a131e82-db54-474d-829e-1e4582eed27d" />
These are optional if you connect through the UI instead.

| Variable | Required | Description |
|---|---|---|
| `COPERNICUS_CLIENT_ID` | Yes (or use UI) | Copernicus Data Space OAuth client ID |
| `COPERNICUS_CLIENT_SECRET` | Yes (or use UI) | Copernicus Data Space OAuth client secret |
| `RF_MODEL_PATH` | No | Path to trained RF model (defaults to `./rf_model.pickle`) |
| `trialx_DATA_DIR` | No | Root directory for all data (defaults to `./trialx_data`) |

## How It Works

Based on [Fisser et al. (2022)](https://ui.adsabs.harvard.edu/abs/2022RemS...14.1595F/abstract), adapted for real-time web streaming.

### The Pipeline

```
Sentinel-2 Image (10m resolution, 5-day revisit)
    |
    +-- 1. Feature Stack (7 features per pixel)
    |     Variance of RGB, Normalized ratio R/B, Normalized ratio G/B,
    |     Mean-centered B04, B03, B02, B08
    |
    +-- 2. Random Forest Classification
    |     Each pixel classified as: background, blue, green, or red
    |     Post-process: threshold background confidence at 0.75
    |
    +-- 3. Recursive Object Extraction
    |     Start at blue pixels, grow through green, then red
    |     Validate: all 3 colors present, 3-5 pixel extent
    |     Score: mean_max_prob + mean_prob > 1.2
    |
    +-- 4. Per-Detection Output
          Lat/lon, heading, speed estimate, confidence score
```

### Capabilities and Limits

<img width="1700" height="1075" alt="Image" src="https://github.com/user-attachments/assets/c9c32a92-9fe8-4bc9-b728-9e997096f456" />
You can also compare trends and historical data between areas


**What it does well:**

- Count large vehicles (trucks, buses) on major highways, roughly 70-80% detection rate with the trained model on European motorways
- Track volume trends over weeks and months
- Estimate speed (plus or minus 15 km/h) and heading (plus or minus 22.5 degrees) per detection
- Cover anywhere on Earth with Sentinel-2 imagery
- Process multi-month archives in minutes with parallel analysis

**What it cannot do:**

- Detect cars (they are smaller than one pixel at 10m resolution)
- Distinguish vehicle types. A military convoy looks the same as a line of delivery trucks. A fuel tanker looks the same as a water tanker. You get "large vehicle," nothing more.
- See through clouds (optical satellite limitation)
- Provide real-time monitoring. Sentinel-2 revisits every 5 days, and imagery is available with a delay. This is a trend analysis tool, not a live feed.
- Guarantee uniform accuracy globally. Best results on dark asphalt in clear conditions. Weaker on light-colored or unpaved roads, and in frequently cloudy regions.


## Interesting Targets

### Built-in Validation Sites

Pre-configured from the original S2TruckDetect research:

| Site | Highway | Bbox | Notes |
|---|---|---|---|
| Braunschweig | A7 | `52.25, 10.45, 52.32, 10.55` | Research-grade validation |
| Frankfurt | A3 | `50.05, 8.55, 50.12, 8.65` | High-density corridor |
| Karlsruhe | A5 | `48.95, 8.35, 49.05, 8.45` | Standard benchmark |

### Worth Investigating

Use "Draw AOI" on the map view. Not built-in, but they produce strong results.

**Chokepoints and disruption indicators:**
- Shahid Rajaee Highway, Bandar Abbas, Iran. 70% of Iran's container trade.
- Bandar Abbas to Sirjan Road (Highway 71). Primary northbound artery from Iran's main port.
- Rotterdam A15. Europe's busiest port feeder highway.
- Laredo I-35, Texas. Largest US-Mexico freight crossing.

**Rerouting and diversion signals:**
- Durban N3, South Africa. Captures Cape of Good Hope rerouting traffic.
- Chabahar port roads, Iran. Alternative that bypasses Hormuz. Inverse signal to Bandar Abbas.

**Economic proxies:**
- Mombasa-Nairobi A109, Kenya. Carries nearly all East African imports.
- UAE E11, Abu Dhabi to Dubai. Excellent imaging conditions, high traffic, good accuracy benchmark.
- Gwadar-Quetta M8, Pakistan. CPEC corridor activity.

**Activity pattern monitoring:**
- Access roads to any facility, installation, or zone where changes in vehicle volume over time are a meaningful signal. Historical comparison is the key capability here. A single observation tells you little. A trend over 6 months tells you a lot.

## Project Structure

```
trialx/
+-- trialx.py              # Backend: API + detection engine
+-- rf_model.pickle        # Trained RF model (not in repo)
+-- .env                   # Credentials (not in repo, optional if using UI)
+-- requirements.txt       # Dependencies
+-- frontend/
|   +-- index.html         # Dashboard
|   +-- app.js             # Frontend logic
|   +-- styles.css         # Styling
+-- trialx_data/           # Auto-created
    +-- sentinel_data/
    |   +-- detections/    # Vehicle crop images
    +-- osm_cache/         # OpenStreetMap cache
    +-- sh_cache/          # SentinelHub cache
```

### API

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/analyze` | Run detection on an AOI (streaming progress) |
| `GET` | `/api/roads` | Fetch road network for a bbox |
| `GET` | `/api/sites` | List preset and historical sites |
| `GET` | `/api/feed` | Recent detection alerts |
| `GET` | `/api/analytics/trends` | Daily counts aggregated across missions |
| `GET` | `/api/detections/:id` | Detections for a specific mission |

## Technical Notes

### Resolution

10m pixels. A truck (roughly 18m) spans about 2 pixels. The motion smear extends it to 3-5 pixels, which is enough for reliable detection. Cars (roughly 4.5m) are sub-pixel and invisible. If you need smaller vehicles, the upgrade path is PlanetScope at 3.7m, which uses the same spectral physics but requires a Planet Labs API key.

### Cloud Cover

trialx uses the Sentinel-2 cloud mask to exclude cloudy pixels. Overcast frames show zero detections. That is correct behavior. Focus on the moving average trend rather than individual days.

### Regional Accuracy

Trained on German autobahns. In practice:

- European motorways: roughly 70-80% detection, 5-10% false positives
- Middle East and North Africa: strong, arid, high-contrast roads, rarely cloudy
- South and Southeast Asia: mixed, monsoon season limits usable frames
- Sub-Saharan Africa: good on paved trunk roads, weak on unpaved




## License

MIT.



---

<div align="center">



</div>
