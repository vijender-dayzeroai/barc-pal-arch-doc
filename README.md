# BARCHITECHTURE — SIDA Building Compliance System
## Technical Architecture & Developer Documentation

---

## Table of Contents

1. [What This System Does](#1-what-this-system-does)
2. [Technology Stack](#2-technology-stack)
3. [Project Structure](#3-project-structure)
4. [End-to-End Data Flow](#4-end-to-end-data-flow)
5. [Module Reference](#5-module-reference)
   - [Root Level](#51-root-level-modules)
   - [Routes (Blueprints)](#52-routes--blueprints)
   - [Extractors](#53-extractors)
   - [Rules Engine](#54-rules-engine)
   - [Validators](#55-validators)
   - [Services](#56-services)
   - [Database Layer](#57-database-layer)
   - [Standards](#58-standards)
   - [Utilities](#59-utilities)
   - [Clients](#510-clients)
6. [Compliance Rules Reference](#6-compliance-rules-reference)
7. [Database Schema](#7-database-schema)
8. [Authentication & Security](#8-authentication--security)
9. [Configuration Reference](#9-configuration-reference)
10. [API Endpoints Reference](#10-api-endpoints-reference)
11. [Key Business Logic](#11-key-business-logic)

---

## 1. What This System Does

This is a **building plan compliance validation system** built for SIDA (based on the Uttarakhand Manual For Online Map Submission, Automated Building Plan Scrutiny & Approval System).

Given a DXF/DWG CAD file (or manual inputs), the system:

1. **Parses the CAD file** — reads layers, polylines, dimensions, and text to extract building parameters
2. **Validates layer standards** — checks whether the CAD file uses SIDA-prescribed layer names and color codes (99 standard layers defined)
3. **Runs 13+ compliance checks** — evaluates setbacks, FAR, height, ground coverage, green area, parking, etc. against SIDA regulations stored in a database
4. **Generates a compliance report** — lists violations, warnings, and passed checks with a compliance percentage
5. **Submits to external systems** — optionally POSTs results to an external CAF (Completion Acceptance Form) API when a `caf_id` is provided

The system targets **Uttarakhand building regulations** for plain and hilly terrain, urban and rural areas, across 30+ building types (Residential, Commercial, Industrial, Educational, Health, Tourism, etc.).

---

## 2. Technology Stack

| Component | Technology |
|-----------|------------|
| Backend framework | Python 3.8+, Flask 3.x |
| CAD parsing | ezdxf |
| Geometry engine | Shapely |
| Data processing | pandas, openpyxl |
| Password hashing | bcrypt |
| Database | SQLite (default) / PostgreSQL |
| DWG conversion | ODA File Converter / LibreCAD |
| Server | Flask dev server (gunicorn for production) |

---

## 3. Project Structure

```
barchitechture/
├── app_new.py                        # Entry point — creates app, starts Flask
├── config.py                         # All configuration (Flask, DB, uploads, logging)
├── auth.py                           # Authentication — bcrypt, session, login decorators
├── validators.py                     # Input validation — file, building type, numerics
├── decorators.py                     # Route decorators — error handling, field validation
├── db_connection.py                  # DB abstraction layer (SQLite/PostgreSQL)
├── database_config.py                # DB type configuration
├── requirements.txt
│
├── app/
│   ├── __init__.py                   # Application factory — create_app()
│   │
│   ├── routes/
│   │   ├── main.py                   # Home, help, manual input, examples
│   │   ├── sida.py                   # Core SIDA flow: upload → layers → process → results
│   │   ├── api.py                    # REST API endpoints
│   │   ├── dashboard.py              # Login, overview, profile
│   │   ├── file_history.py           # History list, export, view, download
│   │   └── file_upload.py            # File upload handling
│   │
│   ├── extractors/
│   │   ├── sida_dxf_extractor.py     # Main DXF parameter extractor
│   │   ├── layer_extractor.py        # Layer enumeration and SIDA layer validation
│   │   ├── extract_layer_measurements.py  # Color-based area/length extraction
│   │   ├── setback_converter.py      # Geometric setback distance calculation
│   │   ├── comprehensive_area_calculator.py  # Multi-floor area calculations
│   │   └── shapely_parking_calculator.py     # Parking space calculation
│   │
│   ├── rules/
│   │   └── sida_rules_engine.py      # 13-check compliance engine
│   │
│   ├── validators/
│   │   ├── sida_calculations.py      # Parking, RWH, ECS, green area calculations
│   │   ├── building_validation.py    # Comprehensive building parameter validation
│   │   ├── component_overlap_checker.py  # Geometric overlap detection
│   │   └── amenity_area_validator.py     # Amenity area excess/deficit check
│   │
│   ├── services/
│   │   ├── sida_processor.py         # Main orchestration: file → full compliance result
│   │   ├── file_processor.py         # File allowed check, multithreaded layer extraction
│   │   └── progress_service.py       # In-memory progress tracking for background jobs
│   │
│   ├── database/
│   │   ├── sida_interface.py         # Query regulations DB (FAR, height, setbacks, etc.)
│   │   └── file_history.py           # Store and retrieve processing history
│   │
│   ├── standards/
│   │   └── sida_layer_standards.py   # 99 SIDA layer definitions with color codes
│   │
│   ├── utils/
│   │   ├── helpers.py                # Unit scaling, color conversion, data formatting
│   │   ├── json_utils.py             # DictAsAttr, serialize_for_json()
│   │   ├── file_utils.py             # File path management, temp file cleanup
│   │   └── dwg_converter.py          # DWG → DXF via ODA File Converter
│   │
│   └── clients/
│       └── external_api.py           # POST results to external CAF API
│
├── templates/                        # Jinja2 HTML templates
├── static/                           # CSS, JS, images
├── uploads/                          # Uploaded DXF/DWG files
├── logs/                             # Application logs
├── sida_file_history.db              # SQLite: processing history
└── baseline_jsons/                   # Baseline compliance result JSONs
```

---

## 4. End-to-End Data Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│  USER                                                               │
│  Uploads DXF/DWG  ──OR──  Enters building data manually            │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │  File Validation         │
                    │  - Extension (dwg/dxf)   │
                    │  - Magic number check    │
                    │  - Size limit (16MB)     │
                    │  - Secure filename       │
                    └────────────┬────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │  Layer Confirmation Page │
                    │  - LayerExtractor reads  │
                    │    all layers from DXF   │
                    │  - Match against 99 SIDA │
                    │    layer standards       │
                    │  - Show color-coded      │
                    │    present/missing list  │
                    │  - User confirms         │
                    └────────────┬────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │  Background Thread       │
                    │  (progress tracked via   │
                    │   ProgressService)       │
                    └────────────┬────────────┘
                                 │
          ┌──────────────────────┼──────────────────────┐
          │                      │                      │
┌─────────▼────────┐  ┌──────────▼────────┐  ┌────────▼────────────┐
│ SIDADXFExtractor │  │  LayerExtractor    │  │  DB: Query           │
│ - plot area/width│  │  - layer names     │  │  Regulations         │
│ - setbacks       │  │  - color codes     │  │  (plain/hill tables) │
│ - building height│  │  - entity counts   │  │  - FAR limits        │
│ - floor areas    │  │  - SIDA compliance │  │  - height limits     │
│ - special floors │  │    percentage      │  │  - setback rules     │
│ - green area     │  └──────────┬─────────┘  │  - parking reqs      │
│ - parking areas  │             │             └────────┬────────────┘
└─────────┬────────┘             │                      │
          └──────────────────────┼──────────────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │  SIDABuildingRuleEngine   │
                    │  13 compliance checks:   │
                    │  1. Plot width           │
                    │  2. Site plan scale      │
                    │  3. Building height      │
                    │  4. FAR                  │
                    │  5. Setbacks (4)         │
                    │  6. Floor heights        │
                    │  7. Basement rules       │
                    │  8. Stilt floor          │
                    │  9. Service floor        │
                    │  10. Ground coverage     │
                    │  11. Green area          │
                    │  12. Min plot area       │
                    │  13. Layer compliance    │
                    └────────────┬────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │  Frontend Checks         │
                    │  - Loading/unloading     │
                    │  - Road width            │
                    │  - RWH tank capacity     │
                    │  - Parking spaces & ECS  │
                    │  - Amenity area excess   │
                    │  - Component overlaps    │
                    └────────────┬────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │  Result Aggregation      │
                    │  - violations list       │
                    │  - warnings list         │
                    │  - passed checks list    │
                    │  - compliance % =        │
                    │    passed/total × 100    │
                    └────────────┬────────────┘
                                 │
          ┌──────────────────────┼──────────────────────┐
          │                      │                      │
┌─────────▼────────┐  ┌──────────▼────────┐  ┌────────▼────────────┐
│  Render HTML     │  │  Save to DB        │  │  External API        │
│  Results Page    │  │  file_processing   │  │  (if caf_id present) │
│                  │  │  _history          │  │  POST compliance     │
└──────────────────┘  └────────────────────┘  │  result to CAF       │
                                               └─────────────────────┘
```

---

## 5. Module Reference

### 5.1 Root Level Modules

#### `config.py`
Flask configuration management. Defines `Config` class with all settings.

| Setting | Default | Description |
|---------|---------|-------------|
| `SECRET_KEY` | env or generated | Flask session secret |
| `UPLOAD_FOLDER` | `uploads/` | File upload directory |
| `MAX_CONTENT_LENGTH` | 16MB | Max upload size |
| `ALLOWED_EXTENSIONS` | `{dwg, dxf}` | Permitted file types |
| `DB_TYPE` | `sqlite` | Database backend |
| `SQLITE_FILE_HISTORY_DB` | `sida_file_history.db` | History DB path |
| `SQLITE_REGULATIONS_DB` | `sida_regulations.db` | Regulations DB path |
| `LOG_LEVEL` | `INFO` | Logging level |

---

#### `auth.py`
Session-based authentication with bcrypt.

Key functions:
- `hash_password(password)` → bcrypt hash string
- `verify_password(plain, hashed)` → bool
- `check_dashboard_login(username, password)` → bool
- `get_user_profile(username)` → dict with name, email, role
- `@login_required` — decorator; allows iframe bypass via `caf_id` + `token` query params
- `@login_required_dashboard` — strict session check, no bypass

Dashboard users are stored as environment variables:
```
DASHBOARD_USER_admin=<bcrypt_hash>
DASHBOARD_USER_admin_NAME=Administrator
DASHBOARD_USER_admin_EMAIL=admin@domain.com
DASHBOARD_USER_admin_ROLE=Administrator
```

---

#### `validators.py`
All input validation lives here. Never trusts user input.

Key functions:
- `validate_file_upload(file)` — checks extension, magic number, size, filename
- `validate_building_type(value)` — enum check against 30+ valid types
- `validate_terrain_type(value)` — `Plain` or `Hilly`
- `validate_municipal_limits(value)` — `Urban` or `Rural`
- `validate_numeric(value, min, max)` — bounds-checked float
- `validate_caf_id(value)` — format check for CAF ID strings
- `sanitize_filename(filename)` — strips path traversal, null bytes, dangerous chars

---

#### `decorators.py`
Flask route decorators.

| Decorator | Purpose |
|-----------|---------|
| `@handle_errors` | Catches all exceptions; returns JSON `{error, trace}` in debug |
| `@require_fields(*fields)` | Aborts 400 if JSON body missing required fields |
| `@require_file(param)` | Aborts 400 if multipart file missing |
| `@log_request` | Logs method, path, IP for each request |
| `@validate_json_fields(**validators)` | Per-field custom validator functions |
| `@cleanup_on_error(fn)` | Runs `fn()` cleanup on any exception |

---

#### `db_connection.py`
Unified SQLite/PostgreSQL abstraction.

```python
class DatabaseConnection:
    def connect()
    def disconnect()
    def execute(query, params=())  # auto-converts ? ↔ %s
    def fetchone()
    def fetchall()
    def commit()
    def rollback()

# Module-level helpers
get_db_connection(db_path)      # Returns DatabaseConnection
adapt_query_for_db(query, type) # Rewrites placeholders
get_placeholder(type)           # Returns '?' or '%s'
get_returning_clause(type)      # Returns '' or 'RETURNING id'
```

---

### 5.2 Routes / Blueprints

#### `routes/main.py` — Blueprint: `main`

| Route | Method | Auth | Description |
|-------|--------|------|-------------|
| `/` | GET | No | Home page |
| `/manual_input` | GET | No | Manual building input form |
| `/analyze_manual` | POST | No | Runs `IndustrialBuildingRuleEngine` on form data, renders results |
| `/examples` | GET | No | Example building scenarios |
| `/help` | GET | No | Help documentation |
| `/upload_file_redirect` | GET | No | Redirects to SIDA file upload |

---

#### `routes/sida.py` — Blueprint: `sida`

This is the **core processing blueprint**.

| Route | Method | Auth | Description |
|-------|--------|------|-------------|
| `/sida_rules` | GET | No | SIDA rules info page |
| `/sida_manual_input` | GET | No | Manual SIDA input form |
| `/analyze_sida_manual` | POST | No | Runs DB-backed SIDA analysis on manual input |
| `/sida_file_upload` | GET | Login | DXF/DWG upload form |
| `/sida_layer_confirmation` | POST | Login | Extract layers, show confirmation page |
| `/confirm_and_process_sida` | POST | Login | Start background processing thread |
| `/sida_processing/<id>` | GET | No | Progress page (polls via JS) |
| `/sida_results/<id>` | GET | No | Retrieve and display final results |
| `/sida_database_test` | GET | No | DB testing interface |

**`/sida_layer_confirmation` flow:**
1. Saves uploaded file
2. Calls `LayerExtractor.extract_layers_from_file()`
3. Compares layer names/colors against `SIDALayerStandards` (99 definitions)
4. Calls `extract_measurements_by_color()` — returns per-color area/length totals
5. Calculates layer compliance percentage
6. Renders `sida_layer_confirmation.html` with present/missing layers

**`/confirm_and_process_sida` flow:**
1. Generates `processing_id = uuid4()`
2. Spawns `threading.Thread(target=_process_sida_compliance_full, args=(processing_id, ...))`
3. Returns `processing_id` to client; client polls `/sida_processing/<id>`

**`/sida_results/<id>` flow:**
1. Retrieves stored result from `ProgressService`
2. If `caf_id` present: calls `ExternalAPIClient.submit_drawing_data()`
3. Updates DB with API submission status
4. Renders `sida_results.html`

---

#### `routes/api.py` — Blueprint: `api`

| Route | Method | Auth | Description |
|-------|--------|------|-------------|
| `/api/analyze` | POST | No | Non-SIDA compliance analysis |
| `/sida_api/analyze` | POST | No | SIDA compliance analysis via API |
| `/api/layer_status/<id>` | GET | No | Layer extraction status |
| `/api/sida/browse_regulations` | POST | No | Query regulations by type/terrain |
| `/api/sida/custom_query` | POST | No | Execute SELECT-only SQL |
| `/api/sida/test_compliance` | POST | No | Test compliance against DB |
| `/api/sida/export_sample` | GET | No | Export sample regulations CSV |
| `/api/file_history/metrics` | GET | Login | File processing metrics |
| `/api/file_history` | GET | Login | Paginated file history list |
| `/api/file_history/<id>` | GET | Login | Single file result by ID |

**`/api/sida/custom_query` security:**
- Only `SELECT` statements allowed
- Forbidden keywords: `DROP`, `DELETE`, `INSERT`, `UPDATE`, `ALTER`, `CREATE`, `TRUNCATE`, `EXEC`, `EXECUTE`
- Table names whitelisted: only `plain_regulations`, `hill_regulations`, `building_regulations`
- Max query length: 1000 characters
- Fully parameterized — no string interpolation

---

#### `routes/dashboard.py` — Blueprint: `dashboard`

| Route | Method | Auth | Description |
|-------|--------|------|-------------|
| `/dashboard` | GET | No | Entry — redirects based on session |
| `/dashboard/overview` | GET | Login | Metrics + last 10 files chart |
| `/dashboard/login` | GET/POST | No | Login form |
| `/dashboard/logout` | GET | No | Clear session |
| `/dashboard/profile` | GET | Login | User profile |

---

#### `routes/file_history.py` — Blueprint: `file_history`

| Route | Method | Auth | Description |
|-------|--------|------|-------------|
| `/sida_file_history` | GET | Login | Paginated history list |
| `/sida_file_history/export` | GET | Login | Export to Excel or CSV |
| `/sida_file_history/view/<id>` | GET | Login | Full compliance report |
| `/sida_file_history/download/<id>` | GET | Login | Download original uploaded file |

---

### 5.3 Extractors

#### `extractors/sida_dxf_extractor.py` — `SIDADXFExtractor`

The most complex module. Reads a DXF document (via ezdxf) and extracts all building parameters.

**What it extracts:**

| Parameter | Detection Method |
|-----------|-----------------|
| Plot area | Closed polyline on plot boundary layer (color 7) |
| Plot width | Bounding box of plot boundary polyline |
| Setbacks | Lines on setback layers (colors 4/3/6/2), or geometric distance from building boundary to plot boundary |
| Building height | Dimension entities or TEXT containing height keywords |
| Floor area | Closed polylines on floor-specific layers |
| Total floors | Count of floor-specific layers present |
| Building type | Keyword detection in layer names and TEXT entities |
| Stilt floor | Presence of stilt layer + height dimension |
| Basement | Presence of basement layer + count |
| Service floor | Presence of service floor layer |
| Mumty | Presence of mumty layer |
| Green area | Closed polylines on green area layer |
| Parking area | Closed polylines on parking layers |

**Key methods:**
- `extract_sida_parameters(dxf_path)` → `dict` of all parameters
- `_extract_area_measurements(doc)` → areas from closed polylines by layer
- `_extract_setback_dimensions(doc)` → setback values from dimension entities
- `_detect_building_type(doc)` → keyword scan of layer names and TEXT
- `_calculate_ffl_value(doc)` → floor-to-floor height from dimensions
- `_apply_unit_scaling(value, doc)` → auto-detects mm/cm/m and normalizes to meters

**Performance features:**
- `@lru_cache` on expensive geometric calculations
- Thread-safe via `threading.Lock`
- Metadata tracking: detection method, match score, confidence

---

#### `extractors/layer_extractor.py` — `LayerExtractor`

Enumerates all layers in a DXF file and compares against SIDA standards.

```python
@dataclass
class Layer:
    name: str
    color_index: int
    entity_count: int
    entity_types: List[str]  # ['LWPOLYLINE', 'LINE', 'TEXT', ...]
```

Key methods:
- `extract_layers_from_file(file_path)` → `List[Layer]`
- `validate_against_sida_standards(layers)` → `{present: [...], missing: [...], compliance_pct: float}`
- `create_layer_summary(layers)` → `{total, present_sida, missing_sida, compliance_pct}`

---

#### `extractors/extract_layer_measurements.py`

Extracts total area and total length per AutoCAD color code.

```python
extract_measurements_by_color(doc) → {
    color_index: {
        'count': int,
        'area': float,       # sqm, from closed polylines via Shapely
        'length': float,     # meters, from open polylines
        'entity_types': list
    }
}
```

Uses Shapely `Polygon` for area calculation and `LineString` for length.

---

#### `extractors/comprehensive_area_calculator.py` — `ComprehensiveAreaCalculator`

Advanced multi-floor area calculation.

Calculates:
- Total covered area per floor
- FAR-contributing area (after deductions)
- Deductions: lift shafts, stairwells, fire escape stairs, double-height voids
- Parking areas by type (basement, stilt, open, mechanical)
- Returns structured dict per floor + totals

---

### 5.4 Rules Engine

#### `rules/sida_rules_engine.py` — `SIDABuildingRuleEngine`

The compliance decision engine. Takes a `SIDABuildingData` object, queries the database via `SIDADatabaseInterface`, and runs 13 checks.

**Input — `SIDABuildingData`:**
```python
@dataclass
class SIDABuildingData:
    plot_area: float              # sqm
    plot_width: float             # meters
    building_type: BuildingType   # enum
    plot_type: PlotType           # PLAIN or HILLY
    area_location: AreaLocation   # URBAN or RURAL
    floor_area: float             # sqm per floor
    total_floors: int
    building_height: float        # meters
    road_width: float             # meters
    setback_front: float
    setback_rear: float
    setback_side1: float
    setback_side2: float
    green_area: float             # sqm
    parking_area: float           # sqm
    # Optional special floors
    has_stilt: bool
    stilt_height: float
    has_basement: bool
    basement_count: int
    basement_height: float
    has_service_floor: bool
    service_floor_height: float
    has_mumty: bool
    mumty_height: float
    # Layer compliance data
    layer_data: dict
```

**Output — `SIDAComplianceResult`:**
```python
@dataclass
class SIDAComplianceResult:
    is_compliant: bool
    compliance_percentage: float        # 0–100
    violations: List[str]               # Critical failures
    warnings: List[str]                 # Non-critical issues
    passed_checks: List[str]            # Checks that passed
    detailed_results: Dict[str, dict]   # Per-check details with values vs limits
```

**The 13 checks and their logic:**

| # | Check | Logic |
|---|-------|-------|
| 1 | Plot width | Plain ≥10m, Hilly ≥6m |
| 2 | Site plan scale | Scale recommendation based on plot area size |
| 3 | Building height | `effective_height = total_height - exemptions ≤ allowed_height(road_width, type)` |
| 4 | FAR | `effective_FAR = (total_area - exemptions) / net_plot_area ≤ max_FAR` |
| 5 | Setbacks | Each of front/rear/side1/side2 ≥ required |
| 6 | Floor heights | Habitable ≥2.75m, beam clearance ≥2.45m |
| 7 | Basement | If present: min plot area 2000sqm, max count, height 2.4–4.5m |
| 8 | Stilt floor | If present: height ≥2.4m for FAR/height exemption |
| 9 | Service floor | If present: height ≤2.0m |
| 10 | Ground coverage | `(footprint / plot_area) × 100 ≤ max_coverage%` |
| 11 | Green area | `green_area ≥ plot_area × (min_green_pct / 100)` |
| 12 | Min plot area | `plot_area ≥ min_required_plot_area` |
| 13 | Layer compliance | `mandatory_present / total_mandatory × 100 ≥ threshold` |

**Height exemptions (not counted in effective height):**
- Stilt floor height if ≥2.4m
- Service floor (actual height)
- Mumty (actual height)

**FAR exemptions (not counted in effective floor area):**
- Stilt floor area (if height ≥2.4m)
- Service floor area
- Mezzanine floor area
- Basement area (partial, per regulation)

---

### 5.5 Validators

#### `validators/sida_calculations.py`

Calculation-only functions (no DB queries, pure math):

| Function | Input | Output |
|----------|-------|--------|
| `calculate_parking_requirement(area, type)` | Built-up area sqm, building type | Min parking spaces |
| `calculate_ecs_requirement(spaces, type)` | Spaces, vehicle type | ECS equivalent |
| `calculate_rwh_capacity(area, rainfall)` | Roof area sqm, rainfall mm | Tank capacity litres |
| `calculate_green_area_pct(green, plot)` | Green area sqm, plot area sqm | Percentage |
| `calculate_loading_area(type, floors)` | Building type, floor count | Required loading area sqm |

---

#### `validators/building_validation.py` — `BuildingValidator`

Standard building validation with hardcoded thresholds:

| Parameter | Threshold |
|-----------|-----------|
| Min plot area | 500 sqm |
| Max coverage | 60% |
| Max FAR | 1.5 |
| Max height | 24m |
| Min front setback | 6m |
| Min rear setback | 3m |
| Min side setback | 3m |
| Min parking | 1 space / 100 sqm |
| Min road width | 9m |
| Min green area | 15% |
| Min stilt height | 2.4m |
| Max service floor height | 2.0m |

Returns `ComprehensiveValidationReport` with individual pass/fail per parameter.

---

### 5.6 Services

#### `services/sida_processor.py`

Main orchestration function. This is what runs in the background thread.

```
_process_sida_compliance_full(processing_id, file_path, confirmed_layers,
                               building_params, caf_id=None)
```

Steps:
1. `update_progress(id, 10, "Extracting DXF parameters...")`
2. `SIDADXFExtractor.extract_sida_parameters(file_path)` → raw params
3. `LayerExtractor.extract_layers_from_file(file_path)` → layer list
4. `convert_dxf_to_sida_format(params)` → `SIDABuildingData`
5. `update_progress(id, 40, "Querying regulations...")`
6. `SIDADatabaseInterface.get_regulations_for_building(type, terrain)` → regs
7. `SIDABuildingRuleEngine.check_compliance(building_data)` → engine result
8. `update_progress(id, 60, "Running frontend checks...")`
9. Frontend checks (parking, RWH, loading, road width, green area)
10. `ComponentOverlapChecker.check_overlaps(areas)` → overlap violations
11. `AmenityAreaValidator.validate(amenity_areas)` → amenity results
12. `calculate_actual_compliance(all_violations, all_warnings, all_passed)` → final %
13. `update_progress(id, 80, "Generating report...")`
14. `render_template('sida_results.html', ...)` → result HTML string
15. `file_history.save_result(...)` → DB
16. `store_result(processing_id, result_html, result_data)` → ProgressService
17. `update_progress(id, 100, "Complete")`

---

#### `services/progress_service.py`

Thread-safe in-memory progress store.

```python
update_progress(processing_id, percentage, message, status='processing')
get_progress(processing_id) → {'percentage', 'message', 'status'}
store_result(processing_id, html, data)
get_result(processing_id) → {'html', 'data'}
cleanup_progress(processing_id)
```

Storage is a plain `dict` protected by `threading.Lock`. Results survive until explicitly cleaned up (cleaned on `get_result`).

---

#### `services/file_processor.py`

```python
allowed_file(filename) → bool                           # Check extension
extract_layers_multithreaded(file_path, processing_id)  # Background layer extraction
get_layer_processing_status(processing_id) → dict       # Status + result
cleanup_old_layer_results(max_age_seconds=3600)         # Memory cleanup
convert_dxf_to_sida_format(params) → SIDABuildingData  # Format conversion
```

Uses `ThreadPoolExecutor` with separate pools for layer extraction vs general tasks.

---

### 5.7 Database Layer

#### `database/sida_interface.py` — `SIDADatabaseInterface`

Queries `sida_regulations.db`. All queries are parameterized.

**Tables queried:**
- `plain_regulations` — Rules for flat terrain
- `hill_regulations` — Rules for hilly/mountainous terrain
- `building_regulations` — Unified fallback table

**Key methods:**

```python
get_regulations_for_building(building_type, terrain, municipal_limits) → dict
validate_building_compliance(building_data) → {
    'compliant': bool,
    'compliance_percentage': float,
    'violations': list,
    'warnings': list,
    'requirements': dict,   # What the regulations require
    'provided': dict        # What the building has
}
get_setback_requirements(building_type, terrain) → {front, rear, side1, side2}
get_far_limits(building_type, terrain) → float
get_height_limits(building_type, terrain) → float
get_green_area_requirements(building_type, terrain) → {pct, min_area}
get_plot_area_requirements(building_type, terrain) → {min, max}
get_parking_requirements(building_type, area) → {spaces, ecs}
```

---

#### `database/file_history.py`

SQLite CRUD for `file_processing_history` table.

```python
init_db()                              # Create table + indexes
save_result(filename, building_type, terrain, compliance_pct,
            is_compliant, response_json, caf_id, processing_time) → id
get_history(page, per_page, filters) → (records, total_count)
get_by_id(id) → record_dict           # Includes full response_json
get_metrics() → {total, compliant, non_compliant, avg_time, avg_compliance}
update_external_api_status(id, submitted, response)
```

---

### 5.8 Standards

#### `standards/sida_layer_standards.py` — `SIDALayerStandards`

Defines all 99 SIDA-prescribed layers.

```python
@dataclass
class SIDALayerDefinition:
    s_no: int                   # 1–99
    description: str            # Human-readable name
    color_code: int             # AutoCAD color index
    object_type: str            # POLYLINE, LINE, TEXT, HATCH, etc.
    layer_names: List[str]      # Expected layer name strings in DXF
    is_mandatory: bool          # Must be present for valid submission
    floor_specific: bool        # Applies to specific floors
```

**Selected critical layers:**

| S.No | Description | Color | Object Type | Mandatory |
|------|-------------|-------|-------------|-----------|
| 1 | Plot boundary | 7 | POLYLINE | Yes |
| 2 | Front setback | 4 | POLYLINE | Yes |
| 3 | Rear setback | 4 | POLYLINE | Yes |
| 4 | Side setback 1 | 4 | POLYLINE | Yes |
| 5 | Side setback 2 | 4 | POLYLINE | Yes |
| 6 | Road width | 41 | LINE | Yes |
| 7 | Stilt floor boundary | 3 | POLYLINE | No |
| 8 | Basement boundary | 6 | POLYLINE | No |
| 9 | Ground floor | 2 | POLYLINE | Yes |
| 10–27 | First–Eighteenth floor | 2 | POLYLINE | No |
| 28 | Mumty | specific | POLYLINE | No |
| 29 | Service floor | specific | POLYLINE | No |
| 30 | Green area | specific | HATCH | No |

---

### 5.9 Utilities

#### `utils/json_utils.py`

**`DictAsAttr`** — allows `obj.key` syntax on dict data:
```python
d = DictAsAttr({'plot_area': 500, 'nested': {'height': 10}})
d.plot_area         # → 500
d.nested.height     # → 10
d.missing_area      # → 0  (smart default: _area suffix → 0)
d.missing_found     # → [] (smart default: _found suffix → [])
d.to_dict()         # → plain dict
```

**`serialize_for_json(obj)`** — recursively converts to JSON-safe types:
- `dataclass` → dict
- `Enum` → `.value`
- `datetime` → ISO string
- `numpy` types → Python native
- `bytes` → base64 string
- DXF entity objects → `"<DXF {type}>"`
- Nested dicts/lists handled recursively

---

### 5.10 Clients

#### `clients/external_api.py` — `ExternalAPIClient`

```python
submit_drawing_data(caf_id, token, compliance_result, building_data) → {
    'success': bool,
    'status_code': int,
    'response': dict
}
```

POST payload includes:
- `caf_id` — CAF application identifier
- `building_type`, `terrain`, `municipal_limits`
- `plot_area`, `building_height`, `far`, `setbacks`
- `is_compliant`, `compliance_percentage`
- `violations` list
- Full `result_json`

Authentication: Bearer token in `Authorization` header.
Timeout: 60 seconds.

---

## 6. Compliance Rules Reference

### Height Allowed vs Road Width

| Road Width | Plain Terrain | Hilly Terrain |
|------------|---------------|---------------|
| < 6m | 10m | 7m |
| 6–9m | 12m | 9m |
| 9–12m | 15m | 12m |
| 12–18m | 21m | 15m |
| > 18m | No restriction | 18m |

Height exemptions (subtracted from measured height):
- Stilt floor height (if ≥2.4m)
- Service floor height
- Mumty height

### FAR Limits by Building Type (typical values)

| Building Type | Plain FAR | Hilly FAR |
|---------------|-----------|-----------|
| Residential | 1.5–2.5 | 1.5 |
| Commercial | 2.0–3.0 | 2.0 |
| Industrial | 1.0–1.5 | 1.0 |
| Educational | 1.5 | 1.5 |
| Health | 2.0 | 1.5 |

*Exact values from `sida_regulations.db` tables.*

### Setback Requirements (typical, varies by type)

| Setback | Plain Urban | Plain Rural | Hilly |
|---------|-------------|-------------|-------|
| Front | 6–9m | 4.5m | 4.5m |
| Rear | 3–4.5m | 3m | 3m |
| Side 1 | 3–5m | 2.4m | 2.4m |
| Side 2 | 3–5m | 2.4m | 2.4m |

### Ground Coverage Limits

| Building Type | Max Coverage |
|---------------|-------------|
| Residential | 40–50% |
| Commercial | 50–60% |
| Industrial | 50–60% |

### Green Area Requirements

| Terrain | Min Percentage |
|---------|---------------|
| Plain | 15% |
| Hilly | 20% |

### Basement Rules

| Parameter | Plain | Hilly |
|-----------|-------|-------|
| Min plot area required | 2000 sqm | 3000 sqm |
| Max basements allowed | 3 | 1 |
| Min height | 2.4m | 2.4m |
| Max height | 4.5m | 4.5m |
| Min height above ground | 0.9m | 0.9m |

### RWH (Rainwater Harvesting) Formula
```
Tank capacity (L) = Roof area (sqm) × Annual rainfall (mm) × 0.8 × 0.1
```

### ECS (Equivalent Car Space) Conversion
| Vehicle Type | ECS Value |
|-------------|-----------|
| Car | 1.0 |
| Motorcycle | 0.25 |
| Truck/Bus | 3.0 |
| Cycle | 0.1 |

---

## 7. Database Schema

### `sida_file_history.db` — `file_processing_history`

```sql
CREATE TABLE file_processing_history (
    id                      INTEGER PRIMARY KEY AUTOINCREMENT,
    filename                TEXT NOT NULL,
    processed_at            TEXT,               -- ISO timestamp
    building_type           TEXT,
    sub_classification      TEXT,
    terrain_type            TEXT,
    municipal_limits        TEXT,
    compliance_percentage   REAL,
    is_compliant            INTEGER,            -- 0 or 1
    source                  TEXT,               -- 'web' or 'api'
    response_json           TEXT,               -- Full JSON result blob
    caf_id                  TEXT,
    external_api_submitted  INTEGER,            -- 0 or 1
    external_api_response   TEXT,               -- API response JSON
    processing_time_seconds REAL,
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_processed_at ON file_processing_history(processed_at);
CREATE INDEX idx_building_type ON file_processing_history(building_type);
CREATE INDEX idx_compliance ON file_processing_history(is_compliant);
```

### `sida_regulations.db` — `plain_regulations` / `hill_regulations`

```sql
CREATE TABLE plain_regulations (
    id                  INTEGER PRIMARY KEY,
    building_type       TEXT,
    terrain             TEXT,
    municipal_limits    TEXT,
    min_plot_size       REAL,
    max_plot_size       REAL,
    max_far             REAL,
    max_height          REAL,
    min_road_width      REAL,
    setback_front       REAL,
    setback_rear        REAL,
    setback_side1       REAL,
    setback_side2       REAL,
    max_coverage        REAL,
    min_green_area      REAL,
    parking_required    REAL,
    ecs_required        REAL
);
-- hill_regulations has identical schema
```

---

## 8. Authentication & Security

### Auth Flow
```
1. User visits /dashboard/login
2. POST username + password
3. check_dashboard_login() runs bcrypt.checkpw()
4. On success: session['user'] = username, session['logged_in'] = True
5. @login_required checks session on protected routes
6. Iframe bypass: if request.args has valid caf_id + token → skip session check
```

### Setting Up Dashboard Users
```bash
# Generate hash
python -c "import bcrypt; print(bcrypt.hashpw(b'mypassword', bcrypt.gensalt()).decode())"

# Add to .env
DASHBOARD_USER_admin=$2b$12$...hash...
DASHBOARD_USER_admin_NAME=Administrator
DASHBOARD_USER_admin_EMAIL=admin@domain.com
DASHBOARD_USER_admin_ROLE=Administrator
```

### Security Measures
- File uploads: extension + magic number + size + filename sanitization
- SQL: all queries parameterized; custom query API enforces SELECT-only + table whitelist
- Passwords: bcrypt with cost factor 12
- Sessions: Flask session with `SECRET_KEY`
- Filenames: `werkzeug.secure_filename()` + custom sanitization for path traversal

---

## 9. Configuration Reference

### `.env` file (all variables)
```env
# Flask
SECRET_KEY=change-this-in-production
FLASK_ENV=development
DEBUG=True

# File uploads
UPLOAD_FOLDER=uploads
MAX_FILE_SIZE=16777216

# Database
DB_TYPE=sqlite
SQLITE_FILE_HISTORY_DB=sida_file_history.db
SQLITE_REGULATIONS_DB=sida_regulations.db

# PostgreSQL (if DB_TYPE=postgresql)
PG_HOST=localhost
PG_PORT=5432
PG_DATABASE=sida_db
PG_USER=postgres
PG_PASSWORD=yourpassword

# Dashboard users
DASHBOARD_USER_admin=<bcrypt_hash>
DASHBOARD_USER_admin_NAME=Administrator
DASHBOARD_USER_admin_EMAIL=admin@domain.com
DASHBOARD_USER_admin_ROLE=Administrator

# External API
EXTERNAL_API_URL=https://api.example.com/sida/submit
EXTERNAL_API_TIMEOUT=60

# Logging
LOG_LEVEL=INFO
LOG_FILE=app.log
LOG_DIR=logs
```

---

## 10. API Endpoints Reference

### Analyze Building (SIDA)
```
POST /sida_api/analyze
Content-Type: application/json

{
  "building_type": "Residential",
  "terrain_type": "Plain",
  "municipal_limits": "Urban",
  "plot_area": 1000,
  "plot_width": 20,
  "building_height": 12,
  "floor_area": 400,
  "total_floors": 3,
  "road_width": 9,
  "setback_front": 6,
  "setback_rear": 3,
  "setback_side1": 3,
  "setback_side2": 3,
  "green_area": 150,
  "parking_area": 100
}

Response 200:
{
  "is_compliant": true,
  "compliance_percentage": 92.3,
  "violations": [],
  "warnings": ["Site plan scale: 1:500 recommended"],
  "passed_checks": ["Plot width OK", "FAR within limits", ...],
  "detailed_results": { ... }
}
```

### Browse Regulations
```
POST /api/sida/browse_regulations
Content-Type: application/json

{
  "building_type": "Commercial",
  "terrain": "Plain",
  "municipal_limits": "Urban"
}

Response 200:
{
  "regulations": {
    "max_far": 2.5,
    "max_height": 21,
    "setback_front": 6,
    ...
  }
}
```

### File History
```
GET /api/file_history?page=1&per_page=20
Authorization: Session required

Response 200:
{
  "files": [...],
  "total": 150,
  "page": 1,
  "per_page": 20
}
```

---

## 11. Key Business Logic

### Compliance Percentage Calculation
```python
total_checks = len(violations) + len(warnings) + len(passed_checks)
# Only violations count as failures (warnings do not)
failed = len(violations)
passed = total_checks - failed
compliance_pct = (passed / total_checks) * 100 if total_checks > 0 else 0
is_compliant = (len(violations) == 0)
```

### Unit Auto-Detection (DXF files)
Many DXF files store coordinates in millimeters. The system auto-detects:
```python
# From DXF header $INSUNITS
INSUNITS=4  → millimeters → scale by 0.001
INSUNITS=5  → centimeters → scale by 0.01
INSUNITS=6  → meters → no scale
INSUNITS=0  → unknown → infer from value magnitude
```

### Green Area Base Area Priority
```
1. If setback_area explicitly provided → use it
2. Elif plot_area - ground_coverage_area available → use difference
3. Elif plot_area available → use plot_area
4. Else → skip check with warning
```

### External CAF API Integration
When a user uploads a file via the SIDA iframe integration (embedded in external CAF portal):
1. URL contains `?caf_id=ABC123&token=xyz`
2. `@login_required` detects these params and bypasses session check
3. After compliance analysis, results are POSTed to external API
4. DB records API submission status and response
5. User sees results with CAF submission confirmation

---

*Generated: 2026-03-11 | Version: 2.0 Modular*
