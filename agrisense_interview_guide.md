# AgriSense — Complete Interview Guide (Final Year Project)

## 1. Project Overview (Elevator Pitch)

> **AgriSense** is an AI-powered smart agriculture platform that uses **Sentinel-2 satellite imagery**, **machine learning (XGBoost)**, and **Google Earth Engine** to monitor crop health, analyze soil conditions, predict pest risks, and provide actionable recommendations to farmers — all through a modern web dashboard.

**Problem Statement:** Indian farmers lose ~15-25% of crop yield annually due to late detection of crop stress, pest infestations, and poor soil management. Existing solutions require expensive hardware or expert knowledge.

**Solution:** AgriSense democratizes precision agriculture by combining free satellite data (Sentinel-2) with ML predictions, delivering farm-specific insights through an intuitive web interface.

---

## 2. System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    FRONTEND (Next.js 16)                    │
│  Landing Page │ Dashboard │ Farm Map │ Weather │ Predictions │
│  React 19 + Leaflet Maps + Recharts + Tailwind CSS         │
└────────────────────────┬────────────────────────────────────┘
                         │ HTTP/REST (Axios + JWT)
┌────────────────────────▼────────────────────────────────────┐
│                 BACKEND (Express.js on Node.js)             │
│  Auth │ Farms │ Satellite │ CropHealth │ Soil │ Pest │ ML   │
│  Middleware: JWT Auth │ Express-Validator                    │
│  Services: AnalysisEngine │ WeatherService │ GEE Proxy      │
│  Scheduler: DailyAggregator (node-cron)                     │
└──────┬─────────────────────────────────────┬────────────────┘
       │ Mongoose ODM                        │ HTTP (Axios)
┌──────▼──────────┐                 ┌────────▼────────────────┐
│   MongoDB       │                 │   ML Service (FastAPI)  │
│   7 Collections │                 │   XGBoost Model         │
│   GeoJSON Index │                 │   Google Earth Engine   │
└─────────────────┘                 │   Preprocessing Pipeline│
                                    └─────────────────────────┘
```

**Three-tier microservice architecture:**
1. **Frontend** — Next.js 16 (React 19) SPA with SSR
2. **Backend** — Express.js REST API (port 5000)
3. **ML Service** — Python FastAPI (port 5001)

---

## 3. Tech Stack & Justification

| Layer | Technology | Why chosen |
|-------|-----------|------------|
| Frontend | Next.js 16 + React 19 | SSR for SEO, App Router, fast rendering |
| Maps | Leaflet + react-leaflet | Open-source, GeoJSON polygon drawing |
| Charts | Recharts | Declarative, React-native charting |
| Styling | Tailwind CSS 4 | Rapid UI development, responsive |
| Backend | Express.js | Lightweight, middleware ecosystem |
| Database | MongoDB + Mongoose | GeoJSON native support (2dsphere index), flexible schema for farm data |
| Auth | JWT + bcryptjs | Stateless auth, password hashing with salt |
| ML | XGBoost + Scikit-learn | Gradient boosting = best for tabular data, fast inference |
| ML API | FastAPI | Async, auto-docs (Swagger), Pydantic validation |
| Satellite | Google Earth Engine | Free Sentinel-2 L2A data, cloud computing |
| Weather | OpenWeatherMap API | Free tier, current + 7-day forecast |
| Scheduler | node-cron | Daily data aggregation without external service |

---

## 4. Database Design (MongoDB — 8 Collections)

### 4.1 User
- Fields: `name, email, password (hashed), phone, role (farmer/admin), language, farms[]`
- Password hashed with **bcrypt (salt 10)** via Mongoose `pre('save')` hook
- JWT token generated via instance method `getSignedJwtToken()` (30-day expiry)

### 4.2 Farm
- Fields: `name, userId, geometry (GeoJSON Polygon), area, cropType, irrigationType, soilData, healthScore, alerts[]`
- **GeoJSON Polygon** stores farm boundaries as `[[[lng, lat], ...]]`
- `2dsphere` geospatial index for location-based queries
- Instance method `getCentroid()` calculates center point for weather/satellite API calls
- Supports **soft delete** (`isActive` flag)

### 4.3 SatelliteData
- Stores all **13 Sentinel-2 L2A bands** (B01–B12, B8A) as raw reflectance values
- Computed **vegetation indices**: NDVI, EVI, SAVI, NDWI, NDMI, BSI, MSAVI
- `dataSource` enum: `sentinel-2-l2a | google-earth-engine | csv-fallback | synthetic`

### 4.4 CropHealth
- Fields: `farmId, ndviValue, eviValue, saviValue, ndwiValue, ndmiValue, healthScore (0-100), growthStage, problemZones (GeoJSON), recommendations[], weatherConditions`

### 4.5 SoilAnalysis
- Manual: `ph, nitrogen, phosphorus, potassium, moisture, organicMatter`
- Satellite-derived: `moistureIndex (NDMI), baresoilIndex (BSI), vegetationCover, estimatedMoisture`

### 4.6 PestRisk
- `riskLevel (low/medium/high), confidence, pestTypes[], weatherFactors, preventionTips[], treatments[], validUntil`

### 4.7 Alert
- `type (weather/pest/health/soil), priority (urgent/warning/info), isRead`
- **TTL index** on `expiresAt` — MongoDB auto-deletes expired alerts

### 4.8 DailySnapshot
- **One record per farm per day** — aggregates health, soil, pest, weather data
- **Unique compound index**: `{ farmId: 1, date: -1 }`
- Powers dashboard charts (last 30 days timeline, 8-week trends)

---

## 5. Module-by-Module Breakdown

### Module 1: Authentication & Authorization

**Files:** `routes/auth.js`, `middleware/auth.js`, `models/User.js`

**Flow:**
1. **Register** → Validate (express-validator) → Check duplicate → Hash password (bcrypt) → Save → Generate JWT
2. **Login** → Find user → Compare password → Generate JWT (30-day expiry)
3. **Protect middleware** → Extract `Bearer` token from header → `jwt.verify()` → Attach `req.user` → `next()`
4. **Role-based access** → `authorize('admin')` middleware checks `req.user.role`

**Key Interview Point:** Passwords are **never stored in plaintext**. The `select: false` on the password field means it's excluded from queries by default — only explicitly included with `.select('+password')` during login.

---

### Module 2: Farm Management (CRUD + GeoJSON)

**Files:** `routes/farms.js`, `models/Farm.js`, `components/FarmMap.js`

**Features:**
- **Create**: User draws polygon on Leaflet map → coordinates sent as GeoJSON → stored with `2dsphere` index
- **Read**: Returns all active farms for authenticated user
- **Update**: Partial updates with field validation
- **Delete**: **Soft delete** — sets `isActive: false`, removes from user's `farms[]` array
- **Authorization**: Every endpoint checks `farm.userId === req.user.id` (ownership verification)

**Frontend (FarmMap.js):** Interactive Leaflet map with `@geoman-io/leaflet-geoman-free` for polygon drawing. Users draw farm boundaries directly on the map.

**Key Interview Point:** GeoJSON `2dsphere` index enables MongoDB to perform geospatial queries (e.g., find farms within a radius). The centroid calculation averages all polygon vertices to get the center point for weather API calls.

---

### Module 3: Satellite Data Integration

**Files:** `routes/satellite.js`, `services/geeProxy.js`, `utils/sentinel2Bands.js`, `gee_service.py`

**Data Pipeline:**
```
Farm Centroid → Build 1km² polygon → Call ML Service /gee/fetch
→ GEE fetches Sentinel-2 L2A → Returns 10 band values
→ Backend computes 7 vegetation indices → Stores in MongoDB
```

**Sentinel-2 Bands Used (10 of 13):**
| Band | Wavelength | Purpose |
|------|-----------|---------|
| B2 (Blue) | 490nm | Soil/vegetation discrimination |
| B3 (Green) | 560nm | Green peak reflectance |
| B4 (Red) | 665nm | Chlorophyll absorption |
| B5-B7 | 705-783nm | Red edge (vegetation stress) |
| B8 (NIR) | 842nm | Vegetation structure |
| B8A | 865nm | Narrow NIR |
| B11 (SWIR1) | 1610nm | Vegetation moisture |
| B12 (SWIR2) | 2190nm | Soil/mineral composition |

**7 Computed Vegetation Indices:**
1. **NDVI** = (NIR − Red) / (NIR + Red) — vegetation greenness
2. **EVI** = 2.5 × (NIR − Red) / (NIR + 6×Red − 7.5×Blue + 1) — improved NDVI
3. **SAVI** = (NIR − Red) / (NIR + Red + 0.5) × 1.5 — soil-adjusted
4. **NDWI** = (Green − NIR) / (Green + NIR) — water content
5. **NDMI** = (NIR − SWIR1) / (NIR + SWIR1) — moisture stress
6. **BSI** = (SWIR1 + Red − NIR − Blue) / (SWIR1 + Red + NIR + Blue) — bare soil
7. **MSAVI** = (2×NIR + 1 − √((2×NIR+1)² − 8×(NIR−Red))) / 2 — modified SAVI

**Three-level fallback:**
1. **Google Earth Engine** (live Sentinel-2 L2A, cloud < 30%)
2. **CSV fallback** (averaged historical training data)
3. **Synthetic** (deterministic values seeded from coordinates)

---

### Module 4: Crop Health Analysis

**Files:** `routes/cropHealth.js`, `services/analysisEngine.js`

**Health Score Algorithm (0-100):**
```
Base: 50 points
+ NDVI contribution:    0-35 points (>0.6: +35, >0.4: +25, >0.2: +15)
+ Moisture (NDMI):      0-15 points (>0.2: +15, >0: +10)
- Weather penalty:      -10 if temp >40°C or <5°C
- Humidity penalty:     -5 if humidity >90% or <20%
- Bare soil penalty:    -10 if BSI >0.2
Result: Clamped to [0, 100]
```

**Classification:** ≥75 = Healthy, ≥50 = Moderate, ≥25 = Poor, <25 = Critical

**Problem Detection:** Automatically identifies low NDVI, water stress, bare soil, disease risk conditions and generates **crop-specific recommendations** (wheat, rice, cotton, sugarcane, etc.).

---

### Module 5: Soil Analysis

**Files:** `routes/soilAnalysis.js`, `services/analysisEngine.js`

**Two modes:**
1. **Manual** — Farmer inputs lab results (pH, N, P, K, moisture, organic matter)
2. **Satellite-derived** — Uses NDMI for moisture index, BSI for bare soil, NDVI for vegetation cover

**Soil Health Score (0-100):**
- Optimal moisture (30-70%): +15 pts
- Good vegetation cover (>60%): +10 pts
- Optimal pH (6.0-7.5): +10 pts
- Good organic matter (>3%): +10 pts
- Adequate nitrogen (≥200 kg/ha): +5 pts

**Recommendations:** Context-aware (e.g., "Soil is acidic. Consider liming." if pH < 5.5)

---

### Module 6: Pest Risk Prediction

**Files:** `routes/pestRisk.js`, `services/analysisEngine.js`

**Risk Score Algorithm:**
```
Base: 20
+ Temperature 25-35°C:    +15 (optimal pest breeding)
+ Humidity > 70%:          +15
+ Humidity > 85%:          +10 (additional)
+ Rainfall > 20mm:         +5
+ Low NDVI (<0.3):         +15 (stressed crops = vulnerable)
+ Growth stage flowering:  +10
```

**Crop-specific pest predictions** with probability scores:
- Wheat: Aphids, Rust
- Rice: Stem Borer, Brown Plant Hopper
- Cotton: Bollworm, Whitefly
- Corn: Fall Armyworm, Corn Borer

**Treatment recommendations**: Cultural → Biological → Organic → Chemical (IPM approach)

---

### Module 7: Weather Integration

**Files:** `routes/weather.js`, `services/weatherService.js`

- **OpenWeatherMap API** for real-time weather + 7-day forecast
- **Weather Advisory Engine**: Generates alerts for heat stress, frost risk, fungal risk, heavy rain, drought, wind damage
- **Synthetic fallback**: Generates weather based on latitude (tropical/temperate/continental zones)
- Forecast aggregates 3-hourly API data into daily summaries (min/max/avg temp, total rainfall)

---

### Module 8: Dashboard & Data Aggregation

**Files:** `routes/dashboard.js`, `services/dailyAggregator.js`

**Dashboard API** returns a single aggregated response containing:
- Overview stats (total farms, avg health, unread alerts, farms needing attention)
- Per-farm summaries with latest health, pest, soil, weather data
- 30-day health timeline (from DailySnapshot collection)
- 8-week trend analysis with up/down/stable indicators

**Daily Aggregator (Cron Job):**
- Runs on server startup (5s delay) + daily at midnight UTC
- For each active farm: collects latest CropHealth, SoilAnalysis, PestRisk + fetches current weather
- Upserts into `DailySnapshot` collection (one record per farm per day)
- **Backfills historical data** from existing CropHealth records on first run

---

### Module 9: ML Prediction Service (XGBoost)

**Files:** `ml-service/app.py`, `train.py`, `preprocessing.py`, `gee_service.py`

#### Training Pipeline
1. **Data**: `4Farms_AllBands_AllIndices_2018_2024.csv` (875KB, 908 samples) + `4Farms_SoilData.csv`
2. **Merge**: Join on `farm_id`, aggregate duplicate tiles per date
3. **Feature Engineering** (26 features total):
   - 10 Sentinel-2 bands (B2-B12, B8A)
   - 3 precomputed indices (EVI, NDWI, SAVI)
   - 5 soil properties (clay, nitrogen, pH, sand, SOC)
   - 3 temporal features (month, day_of_year, season)
   - 4 band ratios (B8/B4, B8/B3, B11/B8, red_edge)
   - 1 interaction feature (nitrogen × NDVI)
4. **Target**: NDVI (proxy for vegetation health)
5. **Model**: XGBoost Regressor (300 trees, depth 6, lr 0.05)
6. **Scaling**: StandardScaler on all features

#### Model Performance
| Metric | Value |
|--------|-------|
| Train R² | 0.9996 |
| Test R² | **0.9992** |
| Test RMSE | 0.0057 |
| Test MAE | 0.0023 |
| CV R² (5-fold) | 0.9992 ± 0.0004 |

**Top 3 features**: SAVI (48.2%), B8/B4 ratio (39.9%), nitrogen×NDVI (11.3%)

#### Inference Pipeline
```
Input (bands + soil + month) → Feature Engineering (26 features)
→ StandardScaler.transform() → XGBoost.predict() → NDVI value
→ ndvi_to_health_score() → { score: 0-100, classification, recommendations }
```

#### API Endpoints
| Endpoint | Description |
|----------|-------------|
| `POST /predict` | Predict from pre-fetched band data |
| `POST /predict/polygon` | Full pipeline: GEE fetch → predict |
| `POST /predict/batch` | Batch predictions |
| `POST /gee/fetch` | Raw satellite data (used by Node backend) |
| `GET /model/info` | Model metadata & metrics |
| `GET /gee/status` | GEE connection check |

---

### Module 10: Alert System

**Files:** `routes/alerts.js`, `models/Alert.js`

- Auto-generated by analysis engine when health < 50, pest risk = high, etc.
- Priority levels: urgent, warning, info
- **TTL index** on `expiresAt` — MongoDB automatically deletes expired alerts
- CRUD: List (with filters), mark read, mark all read, delete

---

## 6. Frontend Pages

| Page | Route | Key Features |
|------|-------|-------------|
| Landing | `/` | Hero section, features, how-it-works |
| Login | `/login` | JWT auth, form validation |
| Register | `/register` | User registration with phone |
| Dashboard | `/dashboard` | Overview cards, health timeline, weekly charts, farm summaries |
| Farms | `/farms` | Farm list, map view, CRUD |
| Farm Detail | `/farms/[id]` | Individual farm analytics |
| New Farm | `/farms/new` | Polygon drawing on Leaflet map |
| Predictions | `/predictions` | ML prediction results per farm |
| Weather | `/weather` | Current + 7-day forecast per farm |
| Reports | `/reports` | Detailed analysis reports |
| Advisory | `/advisory` | AI-generated farming advice |
| Alerts | `/alerts` | Notification center |

**Key Components:**
- `Navbar.js` — Responsive nav with auth state
- `FarmMap.js` — Leaflet map with polygon drawing (15KB)
- `DataCharts.js` — Recharts visualizations
- `HealthChart.js`, `HealthTimelineChart.js`, `WeeklyHealthChart.js` — Specialized chart components

---

## 7. Data Flow Diagrams

### Full Analysis Pipeline
```
User clicks "Analyze" on a farm
  → Frontend calls POST /api/satellite/analyze/:farmId
    → Backend gets farm from MongoDB
    → Computes centroid from GeoJSON polygon
    → Calls ML Service /gee/fetch (GEE → CSV → Synthetic fallback)
    → Receives 10 Sentinel-2 band values
    → Computes 7 vegetation indices (sentinel2Bands.js)
    → Stores SatelliteData in MongoDB
    → Fetches weather from OpenWeatherMap
    → Runs analysisEngine:
      → analyzeCropHealth() → health score + problems + recommendations
      → analyzeSoil() → soil metrics + soil health score
      → assessPestRisk() → risk level + pest types + treatments
      → generateAlerts() → creates Alert documents if needed
    → Updates Farm.healthScore
    → Returns complete analysis to frontend
  → Frontend renders dashboard with charts
```

### ML Prediction Flow
```
User clicks "Predict" on a farm
  → Frontend calls POST /api/ml/predict/:farmId
    → Backend gets farm → computes centroid
    → Fetches Sentinel-2 bands via geeProxy
    → Builds soil data from farm's stored soil info
    → Sends { bands, soil, month } to ML Service POST /predict
      → preprocessing.py: prepare_prediction_input()
        → Computes EVI, NDWI, SAVI from bands
        → Adds soil features, temporal features, band ratios
        → Aligns columns with saved feature_columns.joblib
      → scaler.transform() → StandardScaler normalization
      → model.predict() → XGBoost predicts NDVI
      → ndvi_to_health_score() → maps to 0-100 score
    → Returns { predicted_ndvi, health_score, classification, recommendations }
```

---

## 8. Security Measures

| Aspect | Implementation |
|--------|---------------|
| Password | bcrypt with salt factor 10 |
| Auth | JWT (30-day expiry) in Bearer header |
| Password field | `select: false` — excluded from DB queries by default |
| Input validation | express-validator on all POST/PUT routes |
| Ownership check | Every farm route verifies `farm.userId === req.user.id` |
| CORS | Enabled for cross-origin frontend calls |
| Error handling | Global error handler, no stack traces in production |
| Soft delete | Farms use `isActive` flag, not hard delete |
| TTL index | Alerts auto-expire via MongoDB TTL |

---

## 9. Key Interview Q&A

### Q: Why did you choose XGBoost over other algorithms?
**A:** XGBoost excels at tabular data with mixed feature types (spectral bands + soil + temporal). It handles non-linear relationships, provides feature importance rankings, and is computationally efficient for real-time inference. Our model achieves R² = 0.9992 with only 26 features.

### Q: Why MongoDB instead of PostgreSQL?
**A:** MongoDB's native **GeoJSON support** with `2dsphere` indexing is perfect for storing farm polygon boundaries. The flexible schema accommodates varying soil data and satellite band combinations. The document model naturally maps to our hierarchical farm → analysis → snapshot structure.

### Q: How does the system work without Google Earth Engine?
**A:** We implemented a **three-level fallback**: (1) Live GEE data, (2) CSV-based historical data from our training dataset, (3) Deterministic synthetic values seeded from coordinates. This ensures the system is always functional for demonstrations.

### Q: Explain the NDVI calculation and its significance.
**A:** NDVI = (NIR − Red) / (NIR + Red). Healthy vegetation absorbs red light for photosynthesis and reflects near-infrared. NDVI ranges from -1 to 1; agricultural land typically shows 0.3-0.8. Values < 0.2 indicate bare soil or crop stress. Our ML model predicts NDVI from all 10 bands + soil + temporal data, then converts it to a 0-100 health score.

### Q: What happens when a farmer creates a new farm?
**A:** The farmer draws a polygon on the Leaflet map → coordinates are captured as GeoJSON → sent to the backend → stored with a 2dsphere geospatial index → the centroid is computed by averaging all vertices → this centroid is used for all subsequent weather and satellite data API calls.

### Q: How do you ensure data consistency in the DailySnapshot?
**A:** A **unique compound index** `{ farmId: 1, date: -1 }` prevents duplicate snapshots. The aggregator uses `findOneAndUpdate` with `upsert: true` — it updates if today's snapshot exists, or creates it if not. The `toDateKey()` function strips time to midnight UTC for date-only comparison.

### Q: What is the role of the Analysis Engine?
**A:** It's the core business logic layer (`analysisEngine.js`) with four functions: `analyzeCropHealth()` computes a weighted health score from vegetation indices + weather, `analyzeSoil()` merges satellite-derived and manual soil data, `assessPestRisk()` calculates risk from weather + crop vulnerability + NDVI, and `generateAlerts()` creates notifications for critical conditions.

### Q: How does the ML model handle different farms it hasn't seen?
**A:** The model is trained on spectral band values (universal physical measurements), soil properties (generalizable), and temporal features — not farm IDs. Any farm's Sentinel-2 data follows the same spectral physics, so the model generalizes to new locations. The soil features capture location-specific characteristics.

### Q: What is the purpose of feature scaling?
**A:** We use StandardScaler because our features have vastly different scales — Sentinel-2 bands are ~1000-3000, soil values are 0-1000, NDVI is -1 to 1, and month is 1-12. Without scaling, XGBoost's splits would be dominated by large-valued features. The same scaler fitted during training is saved and reused at inference time.

### Q: How does the frontend handle authentication state?
**A:** React Context API (`AuthProvider`) wraps the entire app. On mount, it checks `localStorage` for a JWT token, calls `/api/auth/me` to validate it, and stores the user object in state. An Axios interceptor attaches the token to every request. On 401 responses, it clears the token and redirects to login.

---

## 10. Future Scope (If Asked)

1. **Drone Integration** — Higher resolution imagery for micro-level analysis
2. **SMS/WhatsApp Alerts** — Twilio integration (schema already has `notificationSent` fields)
3. **Multi-language Support** — User model already has `language` field (en, hi, mr, ta, te, kn)
4. **Mobile App** — React Native sharing the same API backend
5. **Crop Yield Prediction** — Extended ML model predicting kg/hectare
6. **IoT Sensor Integration** — Real-time soil moisture from field sensors
7. **Marketplace** — Connect farmers with buyers based on crop predictions
