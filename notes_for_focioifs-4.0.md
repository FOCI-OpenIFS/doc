# Notes for FOCI-OpenIFS with NEMO 4.2.2

## Configurations

|           |    FOCI-LR    |  FOCI-LRV   |
|-----------|:-------------:|------------:|
| Atm. grid |  Tco95 L91    | Tco95 L91   |
| Oce. grid |  ORCA05 L46   | eORCA05 L75 |
| Status    |  Works        | Not set up  | 

![FOCI-LR](foci-lr.png)

## NEMO settings

CPP keys: Use SI3, XIOS, q-coordinates and OASIS.  
```bash
key_si3 key_xios  key_netcdf4 key_nosignedzero  key_qco key_oasis3
```

Namelist: 

`namelist_cfg` (not the whole thing, but the important bits):

```fortran
&namdom
    ! non-linear free surface
    ln_linssh = .false.
    ! 1800s step
    rn_dt = 1800
/
! We should avoid closed seas such as the Caspian
! The Caspian is handled by the OpenIFS lake model (FLake)
! so any coupling over the Caspian is ignored by OpenIFS anyway
! I'm not sure how to turn off closed seas here, 
! so namcfg and namclo need to be investigated
&namcfg
    ln_closea = .false.
/
&namclo
    ln_maskcs = .true.
    ln_mask_csundef = .true.
    ln_clo_rnf = .true.
/
&namsbc
    ! nn_fsbc effectively sets the ice model time step
    ! It must be called each coupling step (1 hr) or 
    ! more often, so we can only choose 1 or 2
    nn_fsbc = 2
    ln_cpl = .true.
    nn_ice = 2
    ! To do: Test and evaluate the effect of embedding sea ice
    ! (i.e. allow ice to exert pressure on sea surface)
    ln_ice_embd = .false.
/
! Coupling settings are the same as for NEMO 3.6 except
! that we now couple albedo as well. 
&namsbc_cpl
    sn_rcv_rnf = 'coupled', 'no', '', '', ''
    sn_rcv_emp = 'conservative', 'no', '', '', ''
    sn_snd_temp = 'oce and ice', 'no', '', '', ''
    sn_snd_alb = 'ice', 'no', '', '', ''
    sn_snd_thick = 'ice and snow', 'no', '', '', ''
    sn_snd_crt = 'oce only', 'no', 'spherical', 'eastward-northward', 'T'
    sn_snd_co2 = 'none', 'no', '', '', ''
    sn_snd_cond = 'none', 'no', '', '', ''
    sn_snd_mpnd = 'none', 'no', '', '', ''
    sn_snd_sstfrz = 'none', 'no', '', '', ''
    sn_snd_wlev = 'none', 'no', '', '', ''
    sn_snd_ttilyr = 'none', 'no', '', '', ''
    sn_rcv_w10m = 'none', 'no', '', '', ''
    sn_rcv_taumod = 'none', 'no', '', '', ''
    sn_rcv_tau = 'oce and ice', 'no', 'spherical', 'eastward-northward',
                 'T'
    sn_rcv_dqnsdt = 'coupled', 'no', '', '', ''
    sn_rcv_qsr = 'conservative', 'no', '', '', ''
    sn_rcv_qns = 'conservative', 'no', '', '', ''
    sn_rcv_cal = 'coupled', 'no', '', '', ''
    sn_rcv_co2 = 'none', 'no', '', '', ''
/
! Free slip
&namlbc
    rn_shlat = 0
/
! EOS 80, but I would suggest testing TEOS-10 in the future
! Note: This requires changes in file_def_nemo-oce.xml
&nameos
    ln_eos80 = .true.
/

! Lateral diffusion 
&namtra_ldf
    ln_traldf_lap = .true.  ! Laplacian
    ln_traldf_iso = .true.
    ln_traldf_msc = .true.
    ! nn_aht_ijk_t = 20 : Constant Aht which scales with grid cell size
    ! nn_aht_ijk_t = 21 : Aht scaled with deformation radius
    ! The default is 20, in which case
    ! aht0 = 1/2  Ud*Ld = 1/2 * 0.024 * 200e3 = 2400 m2/s for ORCA2
    ! which becomes about 660 m2/s for ORCA05 at equator
    !
    ! Casimir suggested to test nn_aht_ijk_t = 21 (deformation based)
    ! with scale factor 1.5 to increase diffusion further. 
    ! 
    ! This should certainly be evaluated in the future
    nn_aht_ijk_t = 20
    rn_ud = 0.024
    rn_aht_scale = 1.5
/
&namtra_eiv
    ! eddy-induced velocities
    ln_ldfeiv = .true.
    nn_aei_ijk_t = 21
    rn_ue = 0.03
    ln_ldfeiv_dia = .true.
    ! scale by 0.5 as suggested by Gurvan for ORCA05
    rn_aeiv_scale = 0.5
/
! Turn off MLE for now. But should be tested. 
&namtra_mle
    ln_mle = .false.
/
! We want vvl and zstar, especially for L75
&nam_vvl
    ln_vvl_zstar = .true.
/

! Vector form and turn on Hollingsworth correction
! I thought this was not so important for ORCA05
! but apprently it is. 
&namdyn_adv
    ln_dynadv_vec = .true.
    nn_dynkeg = 1
/
! With time-varying vertical coordinate (zstar)
! we need to solve horizonal pressure gradient 
! in s-coordinates
&namdyn_hpg
    ln_hpg_sco = .true.
/
! Use time-split (no idea if filtered is better)
&namdyn_spg
    ln_dynspg_ts = .true.
/
! bi-Laplacian Smagorinsky viscosity
! This was found to be good in v3.6 
&namdyn_ldf
    ln_dynldf_blp = .true.
    ln_dynldf_hor = .true.
    nn_ahm_ijk_t = 32
    rn_uv = 0.164
/
! Stick to TKE scheme since I have no idea what to do with OSMOSIS scheme. 
! EVD for convection, but one could test non-penetrative (npc)
! in the future (I found no big diffs in ORCA1)
! or the new mass flux scheme (mfc). 
&namzdf
    ln_zdftke = .true.
    ln_zdfevd = .true.
/
```

## OpenIFS settings

Namelist: 

`fort.4` (not all, but just the important parts)

```fortran
! Coupling to SI3 is different from LIM2 
! so this needs a new setting. 
&NAMFOCICFG
    FOCI_CPL_NEMO_LIM = .false.
    FOCI_CPL_NEMO_SI3 = .true.
/
&NAMMCC
    LMCCEC = .true.
    LMCCIEC = .false.
    ! This takes albedo from SI3 and overwrites 
    ! the climatological albedo in OpenIFS
    LNEMOLIMALB = .true.
    ! This takes ice surface temperature from SI3
    ! and overwrites ice surface temperature in OpenIFS
    ! Otherwise, the heat fluxes over sea ice 
    ! (computed by OpenIFS and sent to SI3)
    ! would be wrong. 
    ! Lead to underestimated Arctic amplification in 
    ! ECMWF reforecasts. 
    LNEMOLIMTEMP = .true.
    ! Reduce sea albedo from 0.06 to 0.045 to reduce
    ! global cold bias. 
    RALBSEAD_NML = 0.045
    ! Use relative wind stress
    LNEMOLIMCUR = .true.
/
&NAERAD
    ! Aerosol concentrations are taken from CAMS climatology 
    ! which leads to too high anthropogenic aerosol conc in
    ! in pre-industrial climate. 
    ! This will re-scale the anthropogenic aerosols from CAMS 
    ! so that they are representative for e.g. year 1850 
    NAERANT_SCALE = 1
    ! Use new solar spectrum (reduce stratosphere temp bias)
    NSOLARSPECTRUM = 1
    ! Use O3 conc from netCDF files (CMIP6 compliance)
    LEPO3RA = .false.
    LCMIP6O3 = .true.
    LEO3VAR = .true.
    NGHGRAD = 21
    ! Which year to take CMIP6 forcing from
    ! If 0, use year of model
    NCMIPFIXYR = 0
    ! Turn on abrupt 4xCO2 (from NCMIPFIXYR)
    LA4XCO2 = .false.
    ! Turn on 1% CO2 / year (from NCMIPFIXYR)
    L1PCTCO2 = .false.
    ! In HighResMIP control runs, the solar forcing 
    ! should be fixed around 1950. 
    LSOLAR1950 = .false.
/
&NAMDYN
    ! activate mass correction
    LMASCOR = .true.
    ! do not conserve dry mass
    ! instead, we conserve total mass (including water)
    LMASDRY = .false.
/
&NAMCT0
    ! apply mass correction each time step
    NFRMASSCON = 1
    ! turn on XIOS (de-activates GRIB output)
    LXIOS = .true.
/
&NAMGFL
    ! The semi-Lagrangian advection in OpenIFS is not conservative
    ! These mass fixers ensure mass conservation of humidity, 
    ! cloud water etc.  
    ! Not doing this can lead to imbalance of 1-2 W/m2. 
    YQ_NL%LGP = .true.   
    YQ_NL%LSP = .false.  
    YL_NL%LGP = .true.   
    YI_NL%LGP = .true.   
    YA_NL%LGP = .true.   
    YO3_NL%LGP = .true.  
    LTRCMFMG = .true.
    YS_NL%LMASSFIX = .true.
    YR_NL%LMASSFIX = .true.
    YI_NL%LMASSFIX = .true.
    YL_NL%LMASSFIX = .true.
    YQ_NL%LMASSFIX = .true.
    YO3_NL%LGP = .true.
/
&NAMARG
    ! 1800s step for Tco95 (100 km)
    UTSTEP = 1800
/
&NAMCLDP
    ! increase cloud diffusion rate
    ! Note: Needs re-tuning for each resolution
    RCLDIFF = 5e-06
    ! Only liquid water in convective clouds below 600 hPa
    ! Helps reduce Southern Ocean bias
    SCLCT_SWITCH = 2
/
&NAMGWWMS
    ! Reduce launch flux from gravity-wave parametrisation by 25%
    ! This should be much higher (-0.95) for higher resolutions
    ! Important to get a good QBO
    GGAUSSB = -0.25
/
&NAMPPC
    ! Set all fluxes to 0 each output step
    ! Otherwise fluxes would be accumlated throughout the 
    ! simulation. 
    LRSACC = .true.
/
```

## XIOS settings

`iodef.xml` (not all, but the important parts):

```xml
      <!-- XIOS takes the size of the domain and tries to work out the optimal buffer size -->
      <!-- Sometimes (with lots of output fields) this might not be enough -->
      <!-- It is often a good idea to increase the buffers by a factor 2 or 3 --> 
      <!-- Also, one should test performance vs memory mode -->
      <variable_group id="buffer" >
        <variable id="optimal_buffer_size" type="string"> performance </variable>
        <variable id="buffer_size_factor"  type="double"> 3.0         </variable>
      </variable_group>
      
      <!-- Note: In new XIOS and NEMO 4, OASIS_ENDDEF is called by -->
      <!-- XIOS and not the model components (as in NEMO 3.6) -->
      <!-- One can see the difference in src/OCE/SBC/cpl_oasis3.F90 in NEMO -->
      <!-- or src/cplng/oasis/cplng_data_mod.F90 in OpenIFS --> 
      <variable_group id="parameters" >
        <variable id="call_oasis_enddef" type="bool">true</variable>
      </variable_group>
```

`context_ifs.xml` contains one very important part: 

```xml
    <!-- 1) NFRHIS and NFRPOS must be the same --> 
    <!-- 2) This is the frequency at which XIOS is called --> 
    <!--    i.e. it sets the shortest output frequency --> 
    <!-- 3) Negative number means hours -->
    <!-- So, in order to output 3hr data, these must be set to --> 
    <!-- -3 or -1. If set to -6 and you request 3hr output, -->
    <!-- the model will simply crash (and its a hard problem to debug) -->
    <variable_group id="fullpos_freq">
      <variable id="nfrhis" name="NFRHIS" type="int"> -6 </variable>
      <variable id="nfrpos" name="NFRPOS" type="int"> -6 </variable>
    </variable_group>
```

## OASIS settings

* Coupling step: 3600s
* Remapping, atm to ocean, scalar: bilinear
* Remapping, atm to ocean, vector: bicubic
* Remapping, ocean to atm, scalar: bilinear
* Remapping, runoff to ocean: locally conserving
* Remapping, calving to ocean: bilinear
* Remapping, atm to runoff mapper: gauswgt



## Development

* Tuning of model. Currently too cold!
* Make new ocean grid, eORCA05 L75
* Configure strait flux calculations (`diadct.F90`)
* Switch to TEOS-10 ? 
* Upgrade to OpenIFS 48r1 ? 



