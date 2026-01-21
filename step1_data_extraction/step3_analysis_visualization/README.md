# Step 3 – Analysis & Visualization

This step analyzes and visualizes the processed irradiance and temperature data.

## Visualizations produced

### 1. Spatial heatmaps
- Heatmaps of:
  - GHI
  - BHI
  - DHI
  - BNI
- Plotted for selected **daytime timestamps**
- Nighttime values (irradiance = 0) are excluded to improve readability

### 2. Time-series analysis
- Hourly mean irradiance time series
- Hourly near-surface air temperature time series
- Temporal variability over July 2023 is analyzed

## Interpretation
- Spatial patterns reflect solar geometry and cloud conditions
- Temporal patterns show clear diurnal cycles
- Differences between irradiance components highlight atmospheric effects
"""
Step 3 - Analysis & Visualization

Why this script?
- The assignment requires at least:
  (1) One map (heatmap)
  (2) One time-series plot
- We produce:
  1) Mean irradiance time series for GHI/BHI/DHI/BNI (hourly)
  2) Temperature time series (hourly mean)
  3) Daytime mean spatial maps (heatmaps) for each irradiance component
  4) Daytime mean temperature map

Outputs:
- Saves plots into: step3_analysis_visualization/figures/
"""

import os
import glob
import pandas as pd
import xarray as xr
import matplotlib.pyplot as plt
import numpy as np

# Folder to save figures
os.makedirs("figures", exist_ok=True)

# -------------------------
# A) Time series plots from merged dataset (hourly)
# -------------------------
df = pd.read_csv("output_merged_dataset_hourly.csv")
df["time"] = pd.to_datetime(df["time"])

# 1) Mean irradiance components vs time
ts_irr = df.groupby("time")[["GHI", "BHI", "DHI", "BNI"]].mean()

plt.figure(figsize=(11,5))
for col in ts_irr.columns:
    plt.plot(ts_irr.index, ts_irr[col], label=col)
plt.xlabel("Time")
plt.ylabel("Irradiance (W/m²)")
plt.title("Hourly mean irradiance components (July 2023)")
plt.grid(True)
plt.legend()
plt.tight_layout()
plt.savefig("figures/timeseries_irradiance_components.png", dpi=200)
plt.show()

# 2) Temperature vs time (spatial mean)
ts_temp = df.groupby("time")["t2m_C"].mean()

plt.figure(figsize=(11,4))
plt.plot(ts_temp.index, ts_temp.values)
plt.xlabel("Time")
plt.ylabel("Temperature (°C)")
plt.title("Hourly mean 2m air temperature (July 2023)")
plt.grid(True)
plt.tight_layout()
plt.savefig("figures/timeseries_temperature_hourly.png", dpi=200)
plt.show()

# -------------------------
# B) Spatial heatmaps using CAMS NetCDF (better than scatter)
# -------------------------
def open_da(keyword):
    """Open CAMS NetCDF and return DataArray for that component."""
    f = [x for x in glob.glob("v4.6_rev2_*_2023_07*.nc") if f"_{keyword}_" in x][0]
    ds = xr.open_dataset(f)
    var = list(ds.data_vars)[0]
    return ds[var]

# Use GHI to define 'daytime' mask (remove night, where values are 0)
da_GHI = open_da("GHI")
day_mask = da_GHI.mean(["latitude", "longitude"]) > 10  # W/m² threshold

print("Day timesteps kept:", int(day_mask.sum().values), "/", int(da_GHI.sizes["time"]))

def plot_daytime_mean_map(keyword, title, fname_out):
    da = open_da(keyword)

    # Daytime average over the month
    da_map = da.sel(time=day_mask).mean("time")

    # Improve contrast: use percentile scaling
    vals = da_map.values[np.isfinite(da_map.values)]
    vmin, vmax = np.percentile(vals, 5), np.percentile(vals, 95)

    plt.figure(figsize=(8,5))
    im = plt.pcolormesh(
        da_map.longitude, da_map.latitude, da_map,
        shading="auto", vmin=vmin, vmax=vmax
    )
    plt.colorbar(im, label=f"{keyword} (W/m²)")
    plt.xlabel("Longitude (deg)")
    plt.ylabel("Latitude (deg)")
    plt.title(title)
    plt.tight_layout()
    plt.savefig(f"figures/{fname_out}", dpi=200)
    plt.show()

plot_daytime_mean_map("GHI", "Daytime mean GHI — July 2023", "map_daytime_mean_GHI.png")
plot_daytime_mean_map("BHI", "Daytime mean BHI — July 2023", "map_daytime_mean_BHI.png")
plot_daytime_mean_map("DHI", "Daytime mean DHI — July 2023", "map_daytime_mean_DHI.png")
plot_daytime_mean_map("BNI", "Daytime mean BNI — July 2023", "map_daytime_mean_BNI.png")

# -------------------------
# C) Temperature map (daytime mean) from temp.nc
# -------------------------
dsT = xr.open_dataset("temp.nc")
daT = dsT["2t"] - 273.15   # Celsius

# Daytime mask for temperature: use same hours as irradiance mask
# Convert CAMS times -> pandas datetime
times_day = pd.to_datetime(da_GHI.time.values)[day_mask.values]

daT_day = daT.sel(time=times_day).mean("time")

plt.figure(figsize=(8,5))
im = plt.pcolormesh(daT_day.lon, daT_day.lat, daT_day, shading="auto")
plt.colorbar(im, label="Temperature (°C)")
plt.xlabel("Longitude (deg)")
plt.ylabel("Latitude (deg)")
plt.title("Daytime mean 2m temperature — July 2023")
plt.tight_layout()
plt.savefig("figures/map_daytime_mean_temperature.png", dpi=200)
plt.show()

print("✅ Figures saved in: figures/")

