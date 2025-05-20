# Vessel Analysis

`VESSEL_METRICS.py` is a command-line tool for analyzing 3D vessel masks. It skeletonizes the mask, builds a graph representation, extracts vessel segments, and computes various morphometric and tortuosity metrics.

---

## Features

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
* **Aggregation** of per-component metrics into a whole-mask summary

---

## Installation

1. Clone this repository or download `VESSEL_METRICS.py`.
2. Ensure you have Python 3.7+.
3. Install dependencies:

```bash
pip install nibabel numpy pandas networkx scikit-image scipy scikit-learn
```

---

## Usage

```bash
python VESSEL_METRICS.py <input_path> [--min_size INT] [--metrics METRIC [METRIC ...]] [--output_folder PATH] [--no_segment_masks]
```

### Arguments

* `<input_path>`: Path to the vessel mask file. Supported formats:

  * NIfTI: `.nii`, `.nii.gz`
  * NumPy: `.npy`, `.npz`

* `--min_size INT`: *(optional, default: 32)* Minimum voxel count for connected components. Small objects are removed before skeletonization.

* `--metrics METRIC [METRIC ...]`: *(optional)* List of metrics to compute. If not specified, **all** metrics are computed. Valid options:

  * `total_length`, `num_bifurcations`, `bifurcation_density`, `volume`
  * `fractal_dimension`, `lacunarity`
  * `geodesic_length`, `avg_diameter`
  * `spline_arc_length`, `spline_chord_length`, `spline_mean_curvature`, `spline_mean_square_curvature`, `spline_rms_curvature`, `arc_over_chord`, `fit_rmse`
  * `num_loops`, `num_abnormal_degree_nodes`

* `--output_folder PATH`: *(optional, default: `./VESSEL_METRICS`)* Directory to save results and intermediate files.

* `--no_segment_masks`: Skip generation and saving of 3D segment masks (saves disk space and speeds up processing).

---

## Output Structure

After running, `<output_folder>` will contain:

```
output_folder/
├── Conn_comp_1/
│   ├── Conn_comp_1_skeleton.nii.gz    # Skeleton mask for component 1
│   └── Segments/
│       ├── Largest endpoint root/
│       │   ├── Segment_1/
│       │   │   ├── Segment_metrics.csv
│       │   │   └── Segment.nii.gz
│       │   └── ...
│       ├── Second largest endpoint root/
│       └── Largest bifurcation root/
├── Conn_comp_2/
│   └── ...
├── all_components_metrics.csv         # Table of metrics for each component
└── Whole_mask_metrics.csv             # Aggregated whole-mask statistics
```

---

## Metric Descriptions & Computation Details

### 1. Skeletonization & Graph Construction

* **Skeletonization**: uses `skimage.morphology.skeletonize` to reduce the binary mask to a one-voxel-thick centerline.
* **Distance map**: computes Euclidean distance transform (`scipy.ndimage.distance_transform_edt`) on the cleaned mask to estimate vessel radius at each voxel.
* **Graph nodes**: each skeleton voxel becomes a node with its 3D coordinate.
* **Graph edges**: connect nodes within a 3×3×3 neighborhood; weight is the Euclidean distance between nodes.
* **Pruning**: detects triangular cycles (`networkx.cycle_basis`) and removes the heaviest edge in each triangle. They appear in occurrence of the bifurctions for how 26-connectivity build edges.

### 2. Connected Components

* Identifies connected components in the pruned graph (`nx.connected_components`). Each component is analyzed separately.

### 3. General Metrics

* **total\_length**: sum of all edge weights in the component.
* **num\_bifurcations**: count of nodes with degree ≥ 3.
* **bifurcation\_density**: `num_bifurcations / total_length`.
* **volume**: approximates vessel volume by treating each edge as a cylinder:

  $\text{volume} = \sum_{(u,v)} \pi \left( \frac{r_u + r_v}{2} \right)^2 \cdot d_{uv},$

  where \$r\_u, r\_v\$ are radii from the distance map and \$d\_{uv}\$ is edge length.

### 4. Structural Metrics

* **num\_loops**: number of independent cycles (`len(nx.cycle_basis)`).
* **num\_abnormal\_degree\_nodes**: nodes with degree > 3.

### 5. Fractal Dimension

* **Box-counting**: uses logarithmically spaced box sizes, counts non-empty boxes for each size, and fits:

  $\log(N(\epsilon)) = D \cdot \log(1/\epsilon) + c.$

  The slope \$D\$ is the fractal dimension.

### 6. Lacunarity

* Builds a grid of cubic boxes of size \$L = \max(\Delta x,\Delta y,\Delta z)/10\$. Computes point counts per box and:

  $\Lambda = \frac{\mathrm{Var}(n)}{[\mathrm{Mean}(n)]^2} + 1.$

### 7. Segment Extraction & Tortuosity

* **Roots**: selects three roots per component:

  1. Endpoint (degree=1) with largest diameter
  2. Endpoint with second-largest diameter
  3. Bifurcation node (degree≥3) with largest diameter (fallback to root 1)

* **Segments**: shortest paths (`nx.shortest_path`) from each root to other endpoints.

* **Segment metrics**:

  * **geodesic\_length**: sum of consecutive node distances.
  * **avg\_diameter**: average \$2r\$ along nodes.
  * **Spline tortuosity**:

    1. Fit cubic B-spline (`splprep`) to segment points.
    2. Reparameterize by arc length for uniform sampling.
    3. Compute first and second derivatives wrt arc length.
    4. Curvature: \$\kappa(s)=|x'(s)\times x''(s)|/|x'(s)|^3\$.
    5. Optionally weight by node frequency.
    6. Metrics:

       * **spline\_arc\_length**: \$\int ds\$.
       * **spline\_chord\_length**: straight distance endpoints.
       * **arc\_over\_chord**: ratio of arc to chord.
       * **spline\_mean\_curvature**: \$\int \kappa(s)/n(s) , ds\$.
       * **spline\_mean\_square\_curvature**: \$\int \[\kappa(s)]^2/\[n(s)]^2 , ds\$.
       * **spline\_rms\_curvature**: \$\sqrt{\frac{1}{L}\int \[\kappa(s)]^2/\[n(s)]^2 , ds}\$.
       * **fit\_rmse**: RMSE between spline and original points.

* **Aggregation**: computes length-weighted averages of curvature metrics for each root.

---

## Examples

```bash
# All metrics with default settings
python vessel_analysis_modular.py data/vessel_mask.nii.gz

# Subset metrics, custom output folder, no masks
python vessel_analysis_modular.py data/mask.npy \
    --metrics total_length volume fractal_dimension \
    --no_segment_masks --output_folder results/

# Higher min component size
python vessel_analysis_modular.py data/mask.nii --min_size 100
```

