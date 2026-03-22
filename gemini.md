# SVT-AV1 Daala Toggle Implementation

Integrated Daala distortion metric as a standalone CLI parameter `--enable-daala`, decoupling it from the specific `--tune 6` path.

## Changes Overview

### CLI and Configuration
- Added `--enable-daala` parameter to `SvtAv1EncApp`.
- Added `enable_daala` boolean to `EbSvtAv1EncConfiguration`.
- Removed `TUNE_DAALA` (formerly tune 6) from the `Tune` enum and all related validation logic.

### Encoder Logic
- **`enc_mode_config.c`**: Sets `ctx->tune_daala_level` based on the `enable_daala` boolean.
- **`product_coding_loop.c`**: Decoupled the `svt_spatial_full_distortion_daala_kernel` calculation from the SSIM Alt Tune (`SSIM_LVL_1`) condition. Daala distortion is now calculated independently if `enable_daala` is active.
### Mode Decision
- Daala distortion is used as a tie-breaker in `product_coding_loop.c` within `tx_type_search` loop.
- **Luma-only**: Daala distortion is computed only for the Y plane. Chroma planes (Cb/Cr) use default SSD distortion.

### Chroma RDO
- **`full_loop.c`**: Chroma Daala distortion (Cb/Cr) has been removed. Chroma planes use SSD distortion exclusively.
- **`rd_cost.c`**: `full_cost_daala` uses `DIST_DAALA` for luma and `DIST_SSD` for chroma planes.

### Temporal Process Layer (TPL)
- Integrated Daala distortion into TPL cost calculation in `src_ops_process.c`, allowing for perceptually-aware temporal RD optimization.

### Activity Masking
- Enabled activity masking in Daala distortion calculations, now togglable via a parameter in the kernel interface.
- **`mode_decision.c`**: Updated `svt_aom_product_full_mode_decision` to handle both Daala and SSIM tie-breakers. 
    - **Logic**: Priority is given to Daala. If there is a tie in Daala cost, it falls back to SSIM (if enabled), and finally to SSD.
- **`rd_cost.c`**: In `svt_aom_full_cost`, `full_cost_daala` is now calculated if `tune_daala_level > 0`.

## Usage
To enable Daala distortion metric with any tune (e.g., SSIM Alt):
```bash
SvtAv1EncApp -i input.y4m -w 1920 -h 1080 --tune 3 --enable-daala 1 -b output.ivf
```

### Inter-Frame Distortions
- **`product_coding_loop.c`**: Calls to `svt_aom_full_cost` and `svt_aom_full_cost_light_pd0` now use `DIST_DAALA` instead of `DIST_SSD` dynamically when `ctx->tune_daala_level >= 3`.
- **`enc_inter_prediction.c`**: `model_rd_for_sb` uses Daala distortion for luma only; chroma planes use default SSD.

### CDEF Integration (`enable_daala >= 3`)
- **`cdef_process.c`**: Added `compute_cdef_dist_daala()` which replaces MSE with Daala perceptual distortion for luma during CDEF filter strength search.
  - Copies each 8x8 block from the source and filtered output into contiguous `uint16_t` temp buffers (handles 8-bit → 16-bit conversion).
  - Calls `svt_aom_od_compute_dist()` per block with `activity_masking=1`.
  - Luma only; chroma continues using MSE.
  - Subsampling is forced to 1 for luma when Daala CDEF is active.

### Determinism Fixes (`enable-daala >= 2`)
- **`mode_decision.c`**: Updated `svt_spatial_full_distortion_daala_kernel` to fix non-determinism issues.
  - Added zero-initialization for `input_16bit` and `recon_16bit` buffers using `svt_memset`.
  - Implemented padding for blocks smaller than 8x8 (e.g., 4x4, 4x8, 8x4) to ensure they are always processed as at least 8x8 by the Daala distortion function, avoiding out-of-bounds reads and uninitialized memory usage.
  - Unified the 8-bit and HBD paths for small blocks to use a consistent local buffer and stride.
- **`daala_dist.c`**: Added `svt_memset` to zero-initialize internal temporary buffers (`e`, `tmp`, `e_lp`) in `svt_aom_od_compute_dist` for additional safety.

