# Vessel Analysis Modular

`vessel_analysis_modular.py` is a command-line tool for analyzing 3D vessel masks. It skeletonizes the mask, builds a graph representation, extracts vessel segments, and computes various morphometric and tortuosity metrics.

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

1. Clone this repository or download `vessel_analysis_modular.py`.
2. Ensure you have Python 3.7+.
3. Install dependencies:

```bash
pip install nibabel numpy pandas networkx scikit-image scipy scikit-learn
```

---

## Usage

```bash
python vessel_analysis_modular.py <input_path> [--min_size INT] [--metrics METRIC [METRIC ...]] [--output_folder PATH] [--no_segment_masks]
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

* **Skeletonization**: Uses `skimage.morphology.skeletonize` to reduce the binary mask to a one-voxel-thick centerline.
* **Distance map**: Computes Euclidean distance transform (`scipy.ndimage.distance_transform_edt`) on the cleaned mask to estimate vessel radius at each voxel.
* **Graph nodes**: Each skeleton voxel becomes a node, storing its 3D coordinate.
* **Graph edges**: Connects nodes within a 3×3×3 neighborhood. Edge weight is the Euclidean distance between node coordinates.
* **Pruning**: Detects all triangular cycles (3-cycles) via `networkx.cycle_basis` and removes the heaviest edge in each triangle to eliminate spurious loops.

### 2. Connected Components

* After pruning, identifies connected components in the graph (`nx.connected_components`). Each component is processed independently.

### 3. General Metrics

* **total\_length**: Sum of all edge weights within the component graph.
* **num\_bifurcations**: Number of nodes with graph degree ≥ 3.
* **bifurcation\_density**: `num_bifurcations / total_length` (avoids division by zero).
* **volume**: Approximates vessel volume by treating each edge as a cylinder:

  $\text{volume} = \sum_{(u,v)} \pi \left( \frac{r_u + r_v}{2} \right)^2 \cdot d_{uv},$

  where \$r\_u\$, \$r\_v\$ are radii from the distance map, and \$d\_{uv}\$ is the edge length.

### 4. Structural Metrics

* **num\_loops**: Number of independent cycles (using `len(nx.cycle_basis)`).
* **num\_abnormal\_degree\_nodes**: Nodes with degree > 3, indicating possible artifacts.

### 5. Fractal Dimension

* **Box-counting method**: Defines logarithmically spaced box sizes. For each box size \$\epsilon\$, counts non-empty boxes covering the 3D point cloud of skeleton nodes. Performs linear regression on

  $\log(N(\epsilon)) \sim D \cdot \log(1/\epsilon)$

  to estimate fractal dimension \$D\$.

### 6. Lacunarity

* Builds a regular grid of cubic boxes of size \$L = \tfrac{\max(\Delta x,\Delta y,\Delta z)}{10}\$. Computes point counts per box, then

  $\Lambda = \frac{\mathrm{Var}(n)}{[\mathrm{Mean}(n)]^2} + 1.$

### 7. Segment Extraction & Tortuosity

* **Roots**: For each component, identifies three root nodes:

  1. Endpoint (degree=1) with largest diameter
  2. Endpoint with second-largest diameter
  3. Bifurcation node (degree≥3) with largest diameter (fallback to first endpoint)

* **Shortest-path segments**: For each root, computes `nx.shortest_path` to all other endpoints. These paths define vessel segments.

* **Segment metrics**:

  * **geodesic\_length**: Sum of Euclidean distances between successive nodes along the segment.
  * **avg\_diameter**: Mean of \$2r\$ over segment nodes, where \$r\$ from distance map.
  * **Spline tortuosity**:

    1. Fit a cubic B-spline (\$s\$-parameterized) through segment points (`scipy.interpolate.splprep`).
    2. Reparameterize by arc length to ensure uniform sampling.
    3. Compute first (\$d\mathbf{x}/ds\$) and second (\$d^2\mathbf{x}/ds^2\$) derivatives.
    4. Curvature: \$\kappa(s) = | \mathbf{x}'(s) \times \mathbf{x}''(s) | / |\mathbf{x}'(s)|^3\$.
    5. Optionally weight curvature by inverse node frequency to adjust for shared paths.
    6. Metrics:

       * **spline\_arc\_length**: Total arc length \$\int ds\$.
       * **spline\_chord\_length**: Straight distance between endpoints of the spline.
       * **arc\_over\_chord**: Ratio of spline arc length to chord length.
       * **spline\_mean\_curvature**: \$\int \kappa(s)/n(s) , ds\$.
       * **spline\_mean\_square\_curvature**: \$\int \[\kappa(s)]^2/\[n(s)]^2 , ds\$.
       * **spline\_rms\_curvature**: \$\sqrt{\dfrac{1}{L}\int \[\kappa(s)]^2/\[n(s)]^2 , ds }\$.
       * **fit\_rmse**: Root-mean-square error between spline evaluation at original points and those points.

* **Aggregation**: Computes length-weighted averages of curvature metrics across all segments from each root.

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

---

## Developer Notes

* Graph pruning removes spurious triangular loops by edge-weight.
* Connected components ensure isolated vessels are analyzed individually.
* Interpolated counts in tortuosity down-weight high-frequency branching regions.

Contributions and issue reports welcome.
