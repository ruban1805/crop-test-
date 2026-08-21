# Crop Decision Support System (DSS)

> AI-powered, MCDM-based crop recommendation system for Tamil Nadu farmers.
> Built as a **Data Structures & Algorithms project** — every algorithmic
> choice is explicit and justified in code comments.

---

## Quick start

```bash
# 1. Python backend
cd crop-dss
pip install -r requirements.txt

# 2. Run decision engine tests (no API keys needed)
python -m pytest tests/ -v

# 3. Start FastAPI server
uvicorn api.main:app --reload --port 8000

# 4. React Native app
cd mobile
npm install
npx expo start
```

---

## Architecture

```
crop-dss/
├── decision_engine/          # Core MCDM pipeline (pure Python, no APIs)
│   ├── crop_database.py      # Hash Map: 20 Tamil Nadu crops
│   ├── decision_matrix.py    # 2D Matrix: crops × criteria
│   ├── decision_tree.py      # Decision Tree: hard elimination
│   ├── ahp.py                # AHP: criteria weight derivation
│   ├── topsis.py             # TOPSIS: suitability scoring
│   ├── heap_ranker.py        # Min-Heap: top-K extraction O(n log k)
│   ├── trie.py               # Trie: crop name autocomplete
│   ├── sliding_window.py     # Sliding Window: weather trend analysis
│   └── engine.py             # Orchestrator: full pipeline
├── api/
│   ├── main.py               # FastAPI endpoints
│   ├── weather_api.py        # NASA POWER API client
│   ├── soil_api.py           # SoilGrids REST API client
│   ├── ndvi_api.py           # ISRO Bhuvan / MODIS NDVI
│   └── image_analysis.py     # OpenCV + TFLite CNN
├── ml/
│   └── train_yield_model.py  # RandomForest / XGBoost yield model
├── db/
│   ├── schema.sql            # PostgreSQL schema
│   └── seed.sql              # Crop seed data
├── mobile/                   # React Native (Expo)
│   ├── App.js
│   └── src/
│       ├── screens/          # HomeScreen, ResultsScreen, DetailScreen, SearchScreen
│       ├── i18n/             # English + Tamil translations
│       └── utils/            # API client, Trie, AsyncStorage cache
└── tests/
    └── test_decision_engine.py   # 40+ unit tests
```

---

## Data Structures used (DSA project requirements)

| Structure | File | Why |
|---|---|---|
| **Hash Map** | `crop_database.py` | O(1) crop profile lookup by name |
| **2D Matrix** | `decision_matrix.py` | Crops × Criteria decision table for TOPSIS |
| **Decision Tree** | `decision_tree.py` | Hard elimination before TOPSIS (hierarchical rules) |
| **Min-Heap** | `heap_ranker.py` | O(n log k) top-K extraction vs O(n log n) full sort |
| **Trie** | `trie.py` + `TrieSearch.js` | O(L) prefix search for autocomplete |
| **Sliding Window** | `sliding_window.py` | O(n) weather trend (moving average, drought streak) |

---

## How the Suitability % is calculated

```
Input: LandProfile (lat/long, soil pH, soil type, water source, temp, rainfall)

Step 1 — Decision Tree elimination
   Hard rules (soil pH ± 0.5, temperature buffer ± 5°C, minimum rainfall checks)
   → Eliminates clearly incompatible crops before scoring

Step 2 — Decision Matrix (2D)
   Score each remaining crop on 6 criteria: [0.0 – 1.0]
     • soil_fit        = pH bell curve + soil type match
     • water_fit       = available water vs crop need ratio
     • temp_fit        = bell curve on temperature range
     • rainfall_fit    = bell curve on rainfall range
     • market_demand   = log-normalised market price
     • expected_yield  = log-normalised midpoint of yield range

Step 3 — AHP weights
   Expert pairwise comparison matrix (6×6) → normalised weights via
   Saaty's AHP. Default weights:
     soil_fit ~31%, water_fit ~22%, rainfall_fit ~22%,
     temp_fit ~16%, market_demand ~6%, expected_yield ~3%
   Consistency Ratio checked < 0.10.

Step 4 — TOPSIS scoring
   1. Euclidean column normalisation of the decision matrix
   2. Weighted normalised matrix = weight × normalised score
   3. Positive Ideal Solution (PIS) = best value per criterion
   4. Negative Ideal Solution (NIS) = worst value per criterion
   5. Euclidean distances d+ (to PIS) and d- (to NIS)
   6. Closeness Ci = d- / (d+ + d-)  ∈ [0, 1]
   7. Suitability % = Ci × 100

Step 5 — ML blend (optional)
   RandomForest/XGBoost trained on data.gov.in yield data
   Final score = 60% TOPSIS + 40% ML predicted yield (normalised)

Step 6 — Min-Heap ranking
   Top-K crops extracted in O(n log k) using explicit min-heap
```

---

## API Endpoints

| Method | Endpoint | Description |
|---|---|---|
| POST | `/analyze-land` | Full pipeline: soil + weather + MCDM |
| GET | `/crop/{name}` | Single crop profile |
| GET | `/crops` | List all crops |
| GET | `/crops/search?q=<prefix>` | Trie autocomplete |
| GET | `/weather-forecast?lat=&lon=` | NASA POWER weather |
| GET | `/soil-data?lat=&lon=` | SoilGrids soil data |
| GET | `/health` | Health check |

### Sample request

```bash
curl -X POST http://localhost:8000/analyze-land \
  -H "Content-Type: application/json" \
  -d '{
    "latitude": 10.8,
    "longitude": 78.7,
    "land_size_ha": 2.5,
    "water_source": "rain_fed",
    "language": "en",
    "top_k": 5
  }'
```

---

## API Keys needed

| API | Key required | How to get |
|---|---|---|
| NASA POWER | **None** | Free, open access |
| SoilGrids (ISRIC) | **None** | Free REST API |
| ISRO Bhuvan/Bhoonidhi | Yes | Register at https://bhuvan.nrsc.gov.in/ (allow 1–2 weeks) |
| IMD | Yes | Apply at https://mausam.imd.gov.in/imd_latest/contents/data_request.php |
| data.gov.in (yield data) | Optional | https://data.gov.in/ — most datasets free |

Set keys as environment variables:
```bash
export BHUVAN_API_KEY="your_key_here"
export IMD_API_KEY="your_key_here"
```

---

## Database setup

```bash
createdb crop_dss
psql -U postgres -d crop_dss -f db/schema.sql
psql -U postgres -d crop_dss -f db/seed.sql
```

Set `DATABASE_URL` env var:
```bash
export DATABASE_URL="postgresql://postgres:password@localhost/crop_dss"
```

---

## Train the ML yield model

```bash
# Download crop yield CSV from data.gov.in into ml/data/crop_yield.csv
# Then:
python ml/train_yield_model.py --model random_forest

# Output: ml/models/yield_model.pkl + ml/models/model_meta.json
```

---

## Running tests

```bash
# All tests — no API calls, no DB needed
python -m pytest tests/test_decision_engine.py -v

# Expected: 40+ tests, all green
```

---

## Mobile app (React Native / Expo)

```bash
cd mobile
npm install
npx expo start

# Scan QR with Expo Go app (Android/iOS)
# Or run in simulator: press 'a' (Android) or 'i' (iOS)
```

Features:
- GPS location detection
- Land photo upload (gallery or camera)
- Offline-first: caches last result in AsyncStorage
- English / Tamil language toggle (in-app)
- Trie-based crop search (works offline)
- Suitability % badge, TOPSIS score breakdown, sowing calendar

---

## Stretch goals (not yet implemented)

- **Crop rotation graph** (Weighted Graph): after primary recommendation, suggest rotation sequences using a crop compatibility weighted graph where edge weight = soil health benefit of growing crop B after crop A.
- **Mandi price trend charts**: use the `market_prices` table + Victory Native to plot 30-day price trends per crop.

---

## Viva / Report notes

**Q: Why AHP and not just fixed weights?**
A: AHP lets domain experts (agronomists) express relative importance via pairwise comparison without needing to agree on exact numbers. It also provides a Consistency Ratio to validate the judgement.

**Q: Why TOPSIS and not a simple weighted sum?**
A: Weighted sum only measures distance from the worst case. TOPSIS measures *both* closeness to the ideal *and* distance from the anti-ideal, giving more nuanced ranking when alternatives are close.

**Q: Why a min-heap for top-K?**
A: Full sort is O(n log n). A min-heap of size k gives O(n log k). With 20 crops this is trivial, but the structure scales correctly as the crop database grows to hundreds of entries.

**Q: Why a Trie for search?**
A: Hash-map prefix search requires scanning all keys O(n·L). A Trie traversal is O(L + output_size) regardless of database size — crucial for a smooth autocomplete experience on a low-powered mobile device.
