# Step 1 – Data Extraction

This step describes how solar irradiance and temperature data were obtained
from the Copernicus Climate Data Store (CAMS and ERA5).

## Data sources
- CAMS gridded solar radiation (GHI, BHI, DHI, BNI)
- ERA5 reanalysis (2 m air temperature)

## Geographic domain
Spain (bounding box):
- North: 41.82
- South: 41.72
- West: -2.56
- East: -2.4

Data were downloaded manually via the CDS web interface
and uploaded to Google Colab for processing.
"""
Step 1 - Data Extraction (manual download)

Why this script?
- The assignment requires extracting irradiance + temperature data from Copernicus.
- Here we assume you downloaded CAMS irradiance NetCDF files and ERA5 temperature NetCDF (temp.nc)
  using the Copernicus web interface (CDS/Copernicus).

Inputs (downloaded manually and placed in the same folder when running):
- CAMS files (NetCDF): GHI, BHI, DHI, BNI for July 2023
- ERA5 temperature file (NetCDF): temp.nc (variable: 2t)

Outputs:
- Prints which files are found (sanity check).
"""

import glob
import os

# 1) List CAMS irradiance NetCDF files
nc_files = sorted(glob.glob("v4.6_rev2_*_2023_07*.nc"))
print("✅ CAMS NetCDF files found:", len(nc_files))
for f in nc_files:
    print(" -", f)

# 2) Check temperature file
print("\nChecking temperature file temp.nc ...")
print("temp.nc exists?", os.path.exists("temp.nc"))

if not nc_files:
    print("\n❌ No CAMS NetCDF files found. Put the 4 .nc files in the same folder before running.")
else:
    print("\n✅ Step 1 OK: CAMS files detected.")

if not os.path.exists("temp.nc"):
    print("❌ temp.nc not found. Download ERA5 2m temperature for July 2023 and name it temp.nc.")
else:
    print("✅ Step 1 OK: temp.nc detected.")
