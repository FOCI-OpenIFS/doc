# Running OpenIFS 43r3

## Further information

* [OpenIFS](https://confluence.ecmwf.int/display/OIFS/OpenIFS+Home)
* [Running FOCI-OpenIFS](README.md)

## Prerequisites

Follow the instructions on how to install ESM-Tools and configure your environment [here](README.md). 

When you reach "Install model", then come back here. 

## Install model

Get the code

```bash
cd esm
mkdir models
cd models
esm_master get-oifs-43r3-v2
``` 

Compile 
```bash
esm_master comp-oifs-43r3-v2
``` 

## Run an experiment 

The following instructions are for the "climate modelling" course in the Climate Physics Master programme at Kiel University. 

An experiment is configured by a runscript in `.yaml` format. 
The experiment time period, scenario, namelist parameters etc are all set by the runscript. 
The runscript `runscripts/oifs/oifs-43r3-climMEMODEL-nesh.yaml` as lots of comments which hopefully explain the settings. 

We make a new experiment `my_first_exp`. You can set the experiment name with the `-e` flag. 
 
```bash
cd 
cd esm/esm_tools/runscripts/oifs/
esm_runscripts oifs-43r3-climMEMODEL-nesh.yaml -e my_first_exp 
```
This should launch a 1-month experiment using forcing from AMIP in CMIP6. 

## Data

OpenIFS will write data to netCDF files via XIOS. 
The variables to store and at which frequency is controlled by the files in `ifs_xml`. 
See XIOS user guide for more info on how to manipulate those files. 

Files are named e.g. `ECE3_1m_19790101_regular_pl.nc`, where `ECE3` is a fixed experiment name, `1m` refers to the output frequency, `19790101` is the start date, `regular` refers to the horizontal grid and `pl` to the vertical grid. 

During post processing (automatic after a successful run), the tag `ECE3` will be replaced by the experiment name set by the `-e` flag in ESM-Tools. Data will also be compressed and chunked using `ncks` (saves approx 50% space). 

Horizontal grid can be either `reduced` (native model grid) or `regular` (XIOS interpolates  to lon-lat grid of your choice). 
Vertical levels can be as follows: 

| Short name | Long name | Levels |
|------------|-----------|--------|
| `sfc`      | Surface   | Single | 
| `pl`       | Pressure levels | 39 levels (1000 hPa to 3 Pa) |
| `ml`       | Model levels | 91 or 137 levels |
| `th`       | Potential temperature levels | 300K, 320K, 360K | 
| `pv`       | Potential vorticity levels | 2e-6 PVU, 3e-6 PVU | 



