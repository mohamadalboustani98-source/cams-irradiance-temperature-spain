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
