# Step 2 – Data Processing & Georeferencing

This step converts raw CAMS and ERA5 files into clean, harmonized datasets
with consistent spatial and temporal coordinates.

## 1. Irradiance processing (CAMS)

- Input data: CAMS gridded solar radiation (NetCDF format)
- Variables:
  - GHI: Global Horizontal Irradiance
  - BHI: Direct Horizontal Irradiance
  - DHI: Diffuse Horizontal Irradiance
  - BNI: Direct Normal Irradiance
- Original temporal resolution: 15 minutes
- Spatial resolution: regular latitude–longitude grid

### Processing steps
- NetCDF files are opened using **xarray**
- Data are converted to tabular format (pandas DataFrame)
- Irradiance values are resampled to **hourly AND monthly mean values**
- Time, latitude, and longitude are preserved for georeferencing

## 2. Temperature processing (ERA5)

- Input data: ERA5 reanalysis (NetCDF format)
- Variable: 2 m air temperature (2t)
- Temporal resolution: hourly
- Temperature is converted from **Kelvin to Celsius**

## 3. Dataset merging

- Irradiance and temperature datasets are merged using:
  - Time
  - Latitude
  - Longitude
- Only matching grid points and timestamps are retained

## Output
- `output_merged_dataset_hourly.csv`
"""
Step 2 - Data Processing & Georeferencing

Why this script?
- CAMS irradiance data is provided as 15-min values on a lat/lon grid.
- ERA5 temperature is hourly on a lat/lon grid.
- We convert CAMS NetCDF -> a clean CSV
- Then resample irradiance to hourly and merge with temperature using common keys:
  (time, latitude, longitude)

Inputs:
- CAMS NetCDF files: GHI/BHI/DHI/BNI
- ERA5 temperature NetCDF: temp.nc (variable: 2t)

Outputs:
- output_data_irradiance.csv (15-min, with lat/lon/time)
- output_temperature_hourly.csv (hourly, with lat/lon/time)
- output_merged_dataset_hourly.csv (hourly merged dataset)
"""

import glob
import xarray as xr
import pandas as pd

# -------------------------
# A) Build output_data_irradiance.csv from CAMS NetCDF files
# -------------------------
def series_from_nc(nc_file, short_name):
    """Read one CAMS NetCDF file, extract the main variable as a pandas Series with (time, lat, lon)."""
    ds = xr.open_dataset(nc_file)
    var = list(ds.data_vars)[0]   # CAMS variable name inside file
    da = ds[var]

    # Mask fill values if present
    fill = da.attrs.get("_FillValue", None)
    if fill is not None:
        da = da.where(da != fill)

    s = da.to_series()
    s.name = short_name
    return s

# Find CAMS files
f_GHI = [f for f in glob.glob("v4.6_rev2_*_2023_07*.nc") if "_GHI_" in f][0]
f_BHI = [f for f in glob.glob("v4.6_rev2_*_2023_07*.nc") if "_BHI_" in f][0]
f_DHI = [f for f in glob.glob("v4.6_rev2_*_2023_07*.nc") if "_DHI_" in f][0]
f_BNI = [f for f in glob.glob("v4.6_rev2_*_2023_07*.nc") if "_BNI_" in f][0]

s_GHI = series_from_nc(f_GHI, "GHI")
s_BHI = series_from_nc(f_BHI, "BHI")
s_DHI = series_from_nc(f_DHI, "DHI")
s_BNI = series_from_nc(f_BNI, "BNI")

df_irr = pd.concat([s_GHI, s_BHI, s_DHI, s_BNI], axis=1).reset_index()
df_irr = df_irr.rename(columns={"latitude": "latitude", "longitude": "longitude"})

# Ensure time is datetime
df_irr["time"] = pd.to_datetime(df_irr["time"])

df_irr.to_csv("output_data_irradiance.csv", index=False)
print("✅ Saved: output_data_irradiance.csv")
print(df_irr.head())

# -------------------------
# B) Temperature from temp.nc (ERA5 hourly)
# -------------------------
ds_temp = xr.open_dataset("temp.nc")

# ERA5 variable name in your file is "2t" (Kelvin)
da_t = ds_temp["2t"]

# Convert to DataFrame
df_temp = da_t.to_dataframe(name="t2m_K").reset_index()

# Rename coords to match irradiance columns
df_temp = df_temp.rename(columns={"lat": "latitude", "lon": "longitude"})

# Convert Kelvin to Celsius
df_temp["t2m_C"] = df_temp["t2m_K"] - 273.15

# Keep only useful columns
df_temp = df_temp[["time", "latitude", "longitude", "t2m_C"]]
df_temp["time"] = pd.to_datetime(df_temp["time"])

df_temp.to_csv("output_temperature_hourly.csv", index=False)
print("✅ Saved: output_temperature_hourly.csv")
print(df_temp.head())

# -------------------------
# C) Resample irradiance to hourly mean (per grid point)
# -------------------------
irr_cols = ["GHI", "BHI", "DHI", "BNI"]

df_irr_hourly = (
    df_irr.set_index("time")
          .groupby(["latitude", "longitude"])[irr_cols]
          .resample("1H")
          .mean()
          .reset_index()
)

print("✅ Irradiance hourly shape:", df_irr_hourly.shape)

# -------------------------
# D) Merge irradiance (hourly) + temperature (hourly)
# -------------------------
df_merged = pd.merge(
    df_irr_hourly,
    df_temp,
    on=["time", "latitude", "longitude"],
    how="inner"
)

df_merged.to_csv("output_merged_dataset_hourly.csv", index=False)

print("✅ Saved: output_merged_dataset_hourly.csv")
print("Merged shape:", df_merged.shape)
print(df_merged.head())

print("\nMissing values per column:\n", df_merged.isna().sum())
