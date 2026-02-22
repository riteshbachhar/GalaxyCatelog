# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**GW Galaxy Catalogue API** — A REST API serving galaxy catalogue data (GLADE+) with astronomical queries, modeled after the UpGLADE infrastructure. Built as a portfolio project for a CNRS gravitational wave research position.

## Tech Stack

- **Language:** Python (Conda environment, see `.vscode/settings.json`)
- **API Framework:** FastAPI + Uvicorn
- **Database:** PostgreSQL 12+ with PostGIS (spatial/cone search queries)
- **ORM:** SQLAlchemy 2.0+
- **Data Source:** `glade_sample.csv` — ~99,783 galaxies from GLADE+ v7.5.4 (tab-separated VizieR export, 43 MB)
- **Testing:** pytest + FastAPI test client

## Commands

Once the project is set up, the expected commands will be:

```bash
# Install dependencies (once requirements.txt exists)
pip install -r requirements.txt

# Run the API server
uvicorn app.main:app --reload

# Load galaxy data into PostgreSQL
python load_data.py

# Run tests
pytest tests/

# Run a single test
pytest tests/test_galaxies.py::test_cone_search -v
```

## Architecture

This is a **pure REST API backend** — stateless, pagination-based, no frontend.

### Planned directory structure (to be built):
```
GalaxyCatelog/
├── app/
│   ├── main.py          # FastAPI app instance, router includes
│   ├── database.py      # SQLAlchemy engine, session factory
│   ├── models.py        # SQLAlchemy ORM models (Galaxy table)
│   ├── schemas.py       # Pydantic response/request models
│   └── routers/
│       └── galaxies.py  # All /galaxies/* endpoints
├── load_data.py         # One-time CSV ingestion into PostgreSQL
├── glade_sample.csv     # Source data (tab-separated, ~99K galaxies)
└── tests/
    └── test_galaxies.py
```

### Database Schema

```sql
CREATE TABLE galaxies (
    id SERIAL PRIMARY KEY,
    glade_id VARCHAR(50),
    name VARCHAR(100),
    ra DOUBLE PRECISION,              -- Right Ascension (degrees, 0–360)
    dec DOUBLE PRECISION,             -- Declination (degrees, -90–90)
    redshift DOUBLE PRECISION,
    luminosity_distance DOUBLE PRECISION,  -- Megaparsecs (Mpc)
    apparent_mag_b DOUBLE PRECISION,
    apparent_mag_k DOUBLE PRECISION,
    source VARCHAR(50),
    created_at TIMESTAMP DEFAULT NOW()
);
-- Indexes needed on: ra, dec, redshift, luminosity_distance
-- PostGIS geometry column needed for cone_search endpoint
```

### API Endpoints

| Endpoint | Method | Key Query Params |
|---|---|---|
| `/health` | GET | — |
| `/galaxies` | GET | `limit`, `offset` |
| `/galaxies/{id}` | GET | — |
| `/galaxies/search` | GET | `redshift_min`, `redshift_max`, `dist_min`, `dist_max`, `limit`, `offset` |
| `/galaxies/cone_search` | GET | `ra`, `dec`, `radius` (degrees), `limit`, `offset` |

### Data File Notes

`glade_sample.csv` is a VizieR export with:
- A multi-line metadata header (lines starting with `#`)
- Tab-separated columns
- A units row and separator row after the header that must be skipped during ingestion
- Key columns: `RAJ2000`, `DEJ2000`, `dL` (luminosity distance), `Bmag`, `Kmag`
- Empty strings represent missing values (convert to `None`/`NULL`)

The existing `load_data.py` skeleton reads this format and inserts into a `galaxies` table — it assumes the DB and table already exist and has no error handling yet.

## Database Setup

```bash
# Create database and enable PostGIS
psql -U postgres -c "CREATE DATABASE glade_api;"
psql -U postgres -d glade_api -c "CREATE EXTENSION postgis;"
```

The database name is `glade_api` (hardcoded in `load_data.py`).
