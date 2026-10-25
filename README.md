# SISTER L2A ISOFIT reflectance bundle

SISTER (Space-based Imaging Spectroscopy and Thermal PathfindER) was JPL's pathfinder workflow for the Surface Biology and Geology (SBG) mission. Its L2A atmospheric correction step, the `sister-isofit` PGE, turns the L1B radiance, location and observation cubes from the SISTER L1 preprocess step into surface reflectance and its uncertainty with ISOFIT (Imaging Spectrometer Optimal FITting): a water vapor presolve, a 6S lookup table run through the sRTMnet emulator, optimal estimation on superpixels, and an empirical line to every pixel. The SISTER V004 products made this way are archived at the ORNL DAAC (dataset 2335).

This repository holds the PGE and the ISOFIT release it calls as git submodules, pinned to the commits the PGE's `install.sh` names:

```sh
git clone --recurse-submodules https://github.com/gchang/sister-isofit-bundle.git
```

| Submodule | Upstream | Commit | Role |
|---|---|---|---|
| `sister-isofit` | https://github.com/sister-jpl/sister-isofit | b8d5ba99eee28adbf31bcbcc1ee03f7aa037744d (tag `3.0.0`) | the PGE: `run.sh`, `generate_runconfig.py`, `sister_isofit.py`, the surface model priors |
| `isofit` | https://github.com/isofit/isofit | 58e673f460c0fe651af243e359a40acd48fa3b92 (tag `v2.9.8`) | ISOFIT; the PGE runs `isofit/isofit/utils/apply_oe.py` from this folder |

ISOFIT's `setup.py` at this tag still says version 2.9.7.1. Nothing inside the submodules is changed. One more Python package comes from PyPI rather than from a submodule: `hy_tools_lite==1.1.1` (the PGE imports it to read and write ENVI files), installed by the same `pip` steps as in `install.sh`.

## Licenses

Each submodule keeps its own license: `sister-isofit` and `isofit` are Apache-2.0. See the `LICENSE` file in each. 6S and sRTMnet (below) come with their own terms from their own sources.

## The /app layout

The PGE was built into a MAAP image whose `/app` folder held both repositories plus two downloaded artifacts, and `sister_isofit.py` hard-codes that layout:

- `sister_isofit.py:174` looks for ISOFIT in the folder next to `sister-isofit`, so the two submodules sit side by side in this bundle.
- `sister_isofit.py:233` sets `SIXS_DIR=/app/6s` (the 6S build).
- `sister_isofit.py:249` passes `--emulator_base=/app/sRTMnet_v120.h5`; ISOFIT reads `sRTMnet_v120_aux.npz` from the same folder.

So the bundle root must be `/app` when the PGE runs (in a container, clone the bundle to `/app`; on a host, make `/app` a link to the bundle root), and the two artifacts go in the bundle root:

```
/app/                        the bundle root
    sister-isofit/           submodule
    isofit/                  submodule
    6s/sixsV2.1              built below
    sRTMnet_v120.h5          downloaded below
    sRTMnet_v120_aux.npz     downloaded below
```

There is no environment variable to point elsewhere: the released code does not read `SISTER_APP_DIR` or any other.

## Software and artifacts

`sister-isofit/install.sh` downloads 6S and sRTMnet from `s3://sister-ops-registry`, which no longer exists. Public copies of both:

- 6S v2.1: the tag `v2.1` of https://github.com/isofit/6S, which keeps the original 2014 6SV2.1 Fortran sources in a `Sixs/` subfolder (https://github.com/isofit/6S/archive/refs/tags/v2.1.tar.gz).
- sRTMnet v120, the emulator ISOFIT itself downloads: https://avng.jpl.nasa.gov/pub/PBrodrick/isofit/sRTMnet_v120.h5 and https://avng.jpl.nasa.gov/pub/PBrodrick/isofit/sRTMnet_v120_aux.npz (5.8 GB together).

The env `install.sh` builds (named `isofit`, Python 3.8), with conda-forge packages, from the bundle root:

```sh
conda create -n isofit -c conda-forge --override-channels python=3.8 gdal=3.6.2 gfortran=13.1.0 make pip
conda activate isofit

# 6S v2.1, built as install.sh builds it
mkdir 6s && cd 6s
curl -fsSL -o 6S-v2.1.tar.gz https://github.com/isofit/6S/archive/refs/tags/v2.1.tar.gz
tar xzf 6S-v2.1.tar.gz --strip-components=1
mv Sixs/* . && rmdir Sixs
sed -i 's/FFLAGS=  $(EXTRA)/FFLAGS=  $(EXTRA) -std=legacy/' Makefile
make
cd ..

# the sRTMnet emulator
curl -fsSL -O https://avng.jpl.nasa.gov/pub/PBrodrick/isofit/sRTMnet_v120.h5
curl -fsSL -O https://avng.jpl.nasa.gov/pub/PBrodrick/isofit/sRTMnet_v120_aux.npz

# ISOFIT and the PGE's packages, as install.sh installs them
pip install -e ./isofit
pip install hy_tools_lite==1.1.1
pip install Pillow==9.2.0
pip install ray==1.9.2
pip install pystac==1.8.4
```

`install.sh` takes GDAL 3.6.2 from the Anaconda defaults channel and `make` from the `anaconda` channel; conda-forge builds are used here. ISOFIT leaves TensorFlow unpinned, and pip settles on TensorFlow 2.11 because of ISOFIT's `protobuf~=3.19.0`.

## Data

- Input: the three L1B datasets from the SISTER L1 preprocess step (bundle `sister-preprocess-bundle`), or the official ones. Each input is a folder named like the files it holds. For the official example (ORNL DAAC dataset 2335, "SISTER Workflow V004", CMR collection `C3114843226-ORNL_CLOUD`, Earthdata login, files under https://data.ornldaac.earthdata.nasa.gov/protected/sister/SISTER_Workflow_V004/data/), put:
  ```
  EXPERIMENTAL-SISTER_AVCL_L1B_RDN_20110513T175417_004/EXPERIMENTAL-SISTER_AVCL_L1B_RDN_20110513T175417_004.{bin,hdr}
  EXPERIMENTAL-SISTER_AVCL_L1B_RDN_20110513T175417_004_LOC/EXPERIMENTAL-SISTER_AVCL_L1B_RDN_20110513T175417_004_LOC.{bin,hdr}
  EXPERIMENTAL-SISTER_AVCL_L1B_RDN_20110513T175417_004_OBS/EXPERIMENTAL-SISTER_AVCL_L1B_RDN_20110513T175417_004_OBS.{bin,hdr}
  ```
  The preprocess step writes exactly these folders under `output/EXPERIMENTAL-SISTER_AVCL_L1B_RDN_20110513T175417_004/`.
- Official L2A output to compare with, from the same place: `EXPERIMENTAL-SISTER_AVCL_L2A_RFL_20110513T175417_004{.bin,.hdr,_UNC.bin,_UNC.hdr}`.

## Running one scene

With the bundle at `/app` and the env active, in a new empty work folder that holds the three input folders:

```sh
conda activate isofit
mkdir work && cd work
# put the three input folders here (see Data)
/app/sister-isofit/run.sh \
    --radiance_dataset EXPERIMENTAL-SISTER_AVCL_L1B_RDN_20110513T175417_004 \
    --location_dataset EXPERIMENTAL-SISTER_AVCL_L1B_RDN_20110513T175417_004_LOC \
    --observation_dataset EXPERIMENTAL-SISTER_AVCL_L1B_RDN_20110513T175417_004_OBS \
    --n_cores 4 --crid 004
```

`--n_cores` defaults to 32; set it to the cores you have. `--segmentation_size` (default 50) and `--experimental` (default `True`, which adds the `EXPERIMENTAL-` prefix) are the other options; `--crid 004` gives the official V004 names. The PGE needs no network.

The products land in `output/EXPERIMENTAL-SISTER_AVCL_L2A_RFL_20110513T175417_004/`, with one subfolder for the reflectance (`..._004/`) and one for the uncertainty (`..._004_UNC/`), each holding `.bin`, `.hdr`, STAC `.json` and, for the reflectance, a browse `.png`. Those two subfolders are the `--reflectance_dataset` and `--uncertainty_dataset` inputs of the next step, `sister-resample` (bundle `sister-resample-bundle`). The `work/` folder (the 6S lookup table and the ISOFIT configs, about 0.7 GB) can be deleted afterwards.

Resources for this scene (83 by 722 pixels at 30 m, 224 bands): about 80 minutes on 4 cores, most of it the 3,960 6S runs of the lookup table, which is rebuilt for every scene (the official run took about 15 minutes on 32 cores); up to about 14 GB of memory while TensorFlow loads the emulator; 6.5 GB for the emulator and 6S plus about 1 GB of work space.

## Known traps

- **`source activate isofit`.** `run.sh` starts with that line. Inside the MAAP image, where an env called `isofit` exists and conda's `activate` script is on `PATH`, it switches to that env. Anywhere else the line fails with `activate: No such file or directory` and, since the script has no `set -e`, carries on with whatever `python` is on `PATH`. Activate the env yourself first. GDAL then also prints `ERROR 1: PROJ: proj_identify: ... share/proj failed` a few times, because the env's PROJ data path is not set; it does not affect the output.
- **`/app` paths.** See "The /app layout": without `/app/6s` and `/app/sRTMnet_v120.h5` ISOFIT fails, and `sister_isofit.py:260` does not check its exit code, so the error shows up later as a missing output file. `SISTER_APP_DIR` is not read by this release.
- **Never rerun an interrupted job in the same folder.** ISOFIT 2.9.8 creates the `_subs_state`, `_subs_uncert` and `_subs_rfl` files under `work/output/` when the inversion starts, and skips the inversion when they exist (`isofit/isofit/utils/apply_oe.py:364-366`). A job killed during the inversion and rerun in the same folder goes straight to the empirical line with mostly empty inputs and writes wrong reflectance in under a minute without any error. Always start again in a new empty folder.
- **`numpy==1.19.2` and `gdal_array`.** Applies to the preprocess env, not this one: the `sister` package pins numpy 1.19.2, which breaks `from osgeo import gdal_array` under a newer conda GDAL. This env resolves numpy 1.24.
- **Reproducibility.** This step gives the same result on rerun up to floating-point noise (the check below matched the official file to 0.0006 reflectance). The later `sister-reflect_correct` step does not (its BRDF fit draws an unseeded random sample).
