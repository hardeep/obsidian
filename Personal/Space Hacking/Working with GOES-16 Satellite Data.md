## What “NOAA/AWS + Satpy” Means

**NOAA/AWS + Satpy** is a modern, hardware-free way to work with real GOES-16 satellite data:

> NOAA generates the data → AWS hosts it publicly → Satpy turns it into usable imagery and arrays

This approach is widely used for weather analysis, atmospheric research, and machine-learning workflows.

---

## 1. NOAA — The Data Source

**NOAA (National Oceanic and Atmospheric Administration)** operates the GOES satellite fleet, including **GOES-16 (GOES-East)**.

NOAA:
- Operates the satellite instruments
- Processes raw sensor readings
- Publishes calibrated scientific products
- Uses NetCDF (`.nc`) as the primary file format

Common data products include:
- Cloud imagery
- Infrared temperature
- Water vapor
- Smoke and dust detection
- Lightning detection (GLM)

Think of NOAA as the **instrument owner and ground-truth provider**.

---

## 2. AWS — Public Distribution Layer

NOAA mirrors GOES data to **Amazon Web Services (AWS)** so anyone can access it freely.

### GOES-16 Public Bucket

```
s3://noaa-goes16/
```

Characteristics:
- Free and anonymous access
- Near-real-time updates (1–5 minute latency)
- Full resolution
- Organized by instrument, product type, date, and scan region

AWS acts purely as **high-availability storage and distribution**.

---

## 3. Satpy — Making the Data Usable

**Satpy** is a Python library designed specifically for satellite data processing.

Satpy handles:
- Reading GOES NetCDF files
- Interpreting spectral bands
- Creating physical RGB composites
- Reprojecting to geographic coordinates
- Exporting images and arrays

Without Satpy, GOES files are just numerical grids.  
With Satpy, they become maps, heat layers, smoke plumes, and ML-ready arrays.

---

## How the Pieces Fit Together

```
GOES-16 (space)
   ↓
NOAA processing
   ↓
AWS S3 public bucket
   ↓
Satpy (local processing)
   ↓
Images, maps, datasets, ML features
```

No antennas. No SDRs. No RF decoding.

---

## Minimal Conceptual Example (Python)

```python
from satpy import Scene
from satpy.find_files_and_readers import find_files

files = find_files(
    platform='goes16',
    sensor='abi',
    product='ABI-L2-CMIPF',
    start_time='2026-01-01 00:00',
    end_time='2026-01-01 00:10'
)

scn = Scene(reader='abi_l2_nc', filenames=files)
scn.load(['C13'])  # Clean IR band (cloud tops, inversions)
scn.show('C13')
```

---

## Why This Stack Is Popular

| Advantage | Why it Matters |
|--------|----------------|
| Free | No licensing or hardware |
| Scientific-grade | Same data NOAA uses |
| Near real time | Operational use possible |
| Automatable | Pipelines, cron jobs, ML |
| Composable | Combine with ground sensors |

---

## Typical Use Cases

- Atmospheric inversion detection
- Smoke plume tracking
- Fog and low cloud identification
- Surface temperature mapping
- Lightning-triggered wildfire alerts
- Feature extraction for machine learning
- Correlating satellite data with ground air-quality sensors

---

## Summary

- NOAA creates and validates the data  
- AWS distributes it publicly and reliably  
- Satpy converts it into usable scientific outputs  

This stack is the fastest way to go from **spaceborne sensors → actionable atmospheric insight** without building hardware.
