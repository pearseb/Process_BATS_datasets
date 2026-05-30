# Process BATS Datasets

This repository is a lightweight processing workflow for building observational climatologies from Bermuda Atlantic Time-series Study (BATS), BIOS-SCOPE, and BAIT datasets.

It is designed for ocean biogeochemical modelling work: taking heterogeneous observational datasets from BCO-DMO and converting them into consistent monthly climatologies on standard depth grids, stored as `xarray.Dataset` objects and optionally exported to NetCDF.

## Purpose

The aim is to create model-comparison-ready observational products for evaluating WOMBAT and related ocean biogeochemical model output at the BATS site.

The workflow standardises datasets that differ in:

- sampling frequency
- depth levels
- variable names
- units
- quality-flag conventions
- temporal coverage
- file structure

Each processing script follows the same basic logic:

1. Download or read a CSV dataset.
2. Parse the sampling date.
3. Extract calendar month.
4. Convert depth to numeric values.
5. Snap observations to a fixed depth grid.
6. Apply quality flags where available.
7. Convert selected variables to numeric values.
8. Average observations by `month × depth`.
9. Return an `xarray.Dataset`.
10. Optionally export the climatology to NetCDF.

## Datasets currently supported

### BATS pigments

Script: `retrieve_BATS_pigments.py`

Processes the BATS HPLC and fluorometric pigment dataset.

Typical variables include:

- chlorophyll pigments
- accessory pigments
- phytoplankton pigment markers such as fucoxanthin, zeaxanthin, peridinin, chlorophyll-a, and phaeopigments

Output:

```python
ds_pigments
```

with dimensions:

```text
month × depth
```

### BATS net primary production

Script: `retrieve_BATS_primaryproduction.py`

Processes the BATS primary productivity dataset from 14C incubation measurements.

Main variable:

- `npp`

If a precomputed production column is available, the script uses it. Otherwise, it can estimate NPP from light-bottle and dark-bottle measurements.

Output units are intended as:

```text
mg C m-3 day-1
```

### BATS zooplankton biomass

Script: `retrieve_BATS_mesoZoo.py`

Processes zooplankton biomass from net tows.

Variables include wet and dry biomass for size fractions:

- 200–500 µm
- 500–1000 µm
- 1000–2000 µm
- 2000–5000 µm
- >5000 µm

The script converts biomass weights to concentrations by dividing by volume filtered.

Output units:

```text
mg m-3
```

### BATS CTD variables

Script: `retrieve_BATS_CTD.py`

Processes the BATS CTD dataset.

Variables currently included:

- oxygen
- temperature
- salinity
- PAR

The `VARIABLES` dictionary can be edited to add additional CTD variables such as fluorescence or beam attenuation.

### BATS bottle data

Script: `retrieve_BATS_bottle.py`

Processes the BATS bottle dataset.

Variables include:

- temperature
- salinity
- oxygen
- DIC
- alkalinity
- nitrate + nitrite
- nitrite
- phosphate
- silicate
- POC
- PON
- POP
- TOC
- TN
- bacterial abundance
- Prochlorococcus
- Synechococcus
- picoeukaryotes
- nanoeukaryotes
- biogenic silica
- lithogenic silica

Quality flags are applied where available. Values flagged as questionable, bad, or missing are masked before climatology construction.

### BIOS-SCOPE biogeochemical survey

Script: `retrieve_BATS_bioscope.py`

Processes the BIOS-SCOPE survey biogeochemical dataset.

Variables include:

- CTD variables
- oxygen
- nitrate
- nitrite
- ammonium
- phosphate
- silicate
- POC
- PON
- DOC
- TDN
- bacterial abundance
- bacterial production from leucine incorporation
- total dissolved amino acids
- individual amino acids

This dataset is useful for microbial-loop model evaluation because it links DOC, DON, bacterial abundance, bacterial production, nutrients, and amino-acid composition.

### BATS sinking particle fluxes

Script: `retrieve_BATS_particlefluxes.py`

Processes BATS Particle Interceptor Trap System fluxes.

Variables include:

- mass flux
- organic carbon flux
- nitrogen flux
- phosphorus flux
- field-blank-corrected organic carbon flux
- field-blank-corrected nitrogen flux

The default output uses the observed sediment-trap depths:

```text
150, 200, 300, 400 m
```

A custom depth grid can also be supplied.

### BAIT dissolved iron and ligand speciation

Script: `retrieve_BATS_iron.py`

Combines two BAIT datasets into one iron climatology:

1. dissolved Fe concentrations and δ56Fe isotope ratios
2. dissolved Fe-binding ligand concentrations and conditional stability constants

Variables include:

- `dissolved_Fe_bottle`
- `delta56Fe_bottle`
- `dissolved_Fe_boat_pump`
- `delta56Fe_boat_pump`
- `dissolved_Fe_spec`
- `ligand1_conc`
- `ligand1_logK`
- `ligand2_conc`
- `ligand2_logK`

This is designed to support iron-cycle evaluation in WOMBAT, especially for representing dissolved Fe availability, isotope constraints, and ligand-mediated Fe complexation.

## Output structure

Each script returns an `xarray.Dataset`.

Typical dimensions:

```text
month: 1–12
depth: standard depth grid
```

Typical variable structure:

```python
<xarray.Dataset>
Dimensions:  (month: 12, depth: N)
Coordinates:
  * month    (month) int64 1 2 3 4 5 6 7 8 9 10 11 12
  * depth    (depth) float64 ...
Data variables:
    variable_1  (month, depth) float64 ...
    variable_2  (month, depth) float64 ...
```

Missing data remain as `NaN`. This is deliberate. It ensures that all variables share the same month-depth grid even when some variables are not measured at all depths or in all months.

## Why this matters for modelling

These climatologies are intended to make BATS observations easier to compare against ocean biogeochemical model output.

For WOMBAT or another model, the basic comparison workflow is:

1. Extract model output near the BATS site.
2. Convert model output to monthly climatologies.
3. Interpolate or regrid the model to the same depth grid as the observations.
4. Compare seasonal cycles and vertical structure for each tracer or rate.
5. Diagnose which model processes need adjustment.

Examples of model-relevant comparisons:

- surface and subsurface chlorophyll structure
- NPP seasonal cycle
- mesozooplankton biomass
- oxygen profiles
- nutrient supply and depletion
- DOC and TDN vertical structure
- bacterial abundance and production
- sinking POC, PON, and POP flux attenuation
- dissolved Fe and ligand structure

## Repository philosophy

The scripts are intentionally simple and explicit rather than over-engineered. Each dataset has its own processor because BCO-DMO datasets often differ in column names, quality flags, date formats, depth conventions, and units.

This makes each script easy to inspect, modify, and debug when a source dataset changes.

## Requirements

Core Python packages:

```text
numpy
pandas
xarray
requests
netCDF4
```

Optional but useful:

```text
matplotlib
scipy
jupyter
```

Install with:

```bash
pip install numpy pandas xarray requests netCDF4 matplotlib scipy jupyter
```

or, using conda/mamba:

```bash
mamba install numpy pandas xarray requests netcdf4 matplotlib scipy jupyter
```

## Notes and cautions

Depth snapping is not interpolation. A sample is assigned to the nearest depth level if it falls within the allowed tolerance. Samples outside that tolerance are discarded.

Quality-flag handling differs by dataset. Most scripts retain flags 1 and 2 and mask worse flags. This is suitable for climatology construction, but individual research applications may need stricter screening.

Some variables use different units across datasets. For example, POC may be reported in micrograms per kilogram in one dataset and flux units in another. Unit conversion should be handled explicitly before cross-dataset comparison.

The climatologies are sparse for datasets with only a few cruises, especially BAIT iron data. A monthly climatology from four seasonal cruises should be interpreted as a seasonal snapshot, not a robust multi-decadal mean.
