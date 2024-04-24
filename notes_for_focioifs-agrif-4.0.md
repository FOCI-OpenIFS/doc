# Notes for FOCI-OpenIFS with NEMO 4.2.2 and AGRIF

## Install new NEMO configuration 

Made a new build, `ICE_AGRIF_CPL`, based on `tests/ICE_AGRIF`. 
Replaced `key_linssh` by `key_qco` and added `key_oasis3`. 

Added to `https://git.geomar.de/foci/src/nemo_config/ice_agrif_cpl`. 

The command `esm_master get-focioifs-agrif-4.0` will get this configuration, but install it as `ORCA05_SI3_COUPLED_AGRIF`. 

Compiling with `arch-ESMTOOLS_generic_oasis_intel.fcm`, but we might want to go with `arch-ESMTOOLS_generic_oasis_intel_agrif.fcm` to use -O1 instead of -O3. 

## Input data

### NEMO

Get `domain_cfg.nc` and `1_domain_cfg.nc` from Sang-Yeob, who has a working ocean-only configuration. 

Need to remove halo points from domain_cfg so that it has dims 720x510. 

Since we have `subbasins.nc` to define basin masks for Atlantic, Pacific and Indian Oceans in ORCA05, AGRIF will expect a file `1_subbasins.nc`. 
Create this file but set all masks to 0 (for now). 

### OASIS

Copy `atmflx`, `atmtau`, `flxatmos` to `atmflx_1` etc. 
Create `sstocean_1` with constant SST at 273 K and all other fields are 0. 

## ESM-Tools

Rename coupling fields, e.g. `1_O_SnwTck` should be `1_OSnwTck`. 
Now added field definitions inside a `choose_nemo.generation` statement. 

## Namelists

### NEMO

In `1_namelist_cfg`

```fortran
&namdyn_hpg
    ln_hpg_zco = .false.
    ln_hpg_sco = .false.
    ln_hpg_zps = .true.
/
...
&namtsd
ln_tsd_init = .false.         !  ocean initialisation
ln_tsd_dmp  = .false.
/
...
&namsbc
nn_fsbc = 1
/
...
&namagrif
    ln_agrif_2way = .true.
    ln_init_chfrpar = .true.
    ln_vert_remap = .false.
    ln_chk_bathy = .true.
    rn_sponge_tra = 600
    rn_sponge_dyn = 600
/
..
&namtra_ldf
    ln_traldf_off = .false.
    ln_traldf_lap = .true.
    ln_traldf_iso = .true.
    nn_aht_ijk_t = 20
    rn_ud = 0.02
    rn_ld = 10000
/
...
&namdyn_ldf
    ln_dynldf_off = .false.
    ln_dynldf_blp = .true.
    ln_dynldf_hor = .false. ! hor does not work with s-coord
    ln_dynldf_lev = .true. 
    nn_ahm_ijk_t = 32
/
```

In `namelist_cfg` 
```fortran
&namdyn_hpg
    ln_hpg_zco = .false.
    ln_hpg_sco = .false.
    ln_hpg_zps = .true.
/
...
&namsbc
    nn_fsbc = 2
/
...
&namagrif
    ln_spc_dyn = .true.
    rn_sponge_tra = 2000
    rn_sponge_dyn = 2000
/
```

Vertical grid is z-star coordinates since `domain_cfg.nc` and `1_domain_cfg.nc` have 
```bash
ln_zco = 0
ln_sco = 0
ln_zps = 1
```
and these settings override what is set in configuration namelists. Model is compiled with `key_qco`. 
`namelist_ref` has `ln_vvl_zstar = .true.`. 

