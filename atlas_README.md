# Vessel Analysis

`VESSEL_METRICS.py` is a command-line tool for analyzing 3D vessel masks. It skeletonizes the mask, builds a graph representation, extracts vessel segments, and computes various morphometric and tortuosity metrics. **All metrics are computed regionally via atlas-based parcellation, enabling spatially-resolved vessel characterization.**

---

## Features

* **Atlas-based regional analysis**: vessel metrics computed within atlas-defined regions
* **Skeletonization** of a 3D binary vessel mask
* **Graph construction** from skeleton voxels
* **Pruning** of triangular loops
* **Connected component** analysis
* **Extraction of vessel segments** via shortest paths from key root points
* **Computation of metrics** per component and per segment:

  * **General metrics**: total length, bifurcation count & density, volume
  * **Structural metrics**: number of loops, abnormal-degree nodes
  * **Fractal analysis**: fractal dimension via box-counting
  * **Lacunarity**: spatial heterogeneity measure
  * **Tortuosity metrics**: geodesic vs chord length, curvature-based measures
* **Saving outputs**: reconstructed skeletons, segment masks, CSV tables
* **Aggregation** of per-region and per-component metrics into detailed and summary CSV reports

---

## Installation

1. Clone this repository or download `VESSEL_METRICS.py`.
2. Install dependencies (check `requirements.txt`).

---

## Usage

bash
python VESSEL_METRICS.py <mask_path> <atlas_path> [--metrics METRIC [METRIC ...]] [--output_folder PATH] [--no_segment_masks] [--no_conn_comp_masks]



### Arguments

- `<mask_path>`: Path to the vessel mask file. Supported formats:
  - NIfTI: `.nii`, `.nii.gz`
  - NumPy: `.npy`, `.npz`

- `<atlas_path>`: Path to the atlas file (NIfTI `.nii` or `.nii.gz`) used for regional parcellation

- `--metrics METRIC [METRIC ...]` *(optional)*: List of metrics to compute. If omitted, all are computed. Valid metrics:
  - `total_length`, `num_bifurcations`, `bifurcation_density`, `volume`
  - `fractal_dimension`, `lacunarity`
  - `geodesic_length`, `avg_diameter`
  - `spline_arc_length`, `spline_chord_length`, `spline_mean_curvature`
  - `spline_mean_square_curvature`, `spline_rms_curvature`, `arc_over_chord`, `fit_rmse`
  - `num_loops`, `num_abnormal_degree_nodes`

- `--output_folder PATH` *(optional, default: `./ATLAS_VESSEL_METRICS`)*: Directory to save results and intermediate files

- `--no_segment_masks`: Disable saving of 3D segment masks (saves disk space)

- `--no_conn_comp_masks`: Disable saving of connected component reconstructed skeletons


## Output Structure

After running the tool, the specified `output_folder` will contain the following structure:

- `Region_1/`
  - `Conn_comp_1/`
    - `Conn_comp_1_skeleton.nii.gz` – Skeleton mask for component 1
    - `Segments/`
      - `Largest endpoint root/`
        - `Segment_1/`
          - `Segment_metrics.csv`
          - `Segment.nii.gz`
        - *(other segments...)*
      - `Second largest endpoint root/`
      - `Largest bifurcation root/`
  - `Conn_comp_2/`
  - *(other components...)*
- `Region_2/`
  - *(same structure as above...)*
- `all_components_by_region.csv` – Detailed metrics per component grouped by region
- `region_summary.csv` – Aggregated summary of each region, including weighted tortuosity metrics

**Notes:**
- Segment masks are saved unless `--no_segment_masks` is specified.
- Component skeletons are saved unless `--no_conn_comp_masks` is specified.
- The CSV outputs contain geometric, structural, and tortuosity metrics. Tortuosity values in `region_summary.csv` are weighted by total vessel length.
