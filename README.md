# Vessel Analysis

`VESSEL_METRICS.py` is a command-line tool for analyzing 3D vessel masks.  
It skeletonizes the mask, builds a graph representation, extracts vessel segments,  
and computes various morphometric and tortuosity metrics.

---

## Features

* **Skeletonization** of a 3D binary vessel mask  
* **Graph construction** from skeleton voxels  
* **Pruning** of triangular loops  
* **Connected component** analysis  
* **Extraction of vessel segments** via shortest paths from 3 different key root points  
* **Computation of metrics** per component and per segment:  
  * **General metrics**: total length, bifurcation count & density, volume, average diameter  
  * **Structural metrics**: number of loops, number of abnormal-degree nodes  
  * **Fractal analysis**: fractal dimension via box-counting  
  * **Lacunarity**: spatial heterogeneity measure  
  * **Tortuosity metrics**: curvature-based measures  
* **Saving outputs**: reconstructed skeletons, segment masks, CSV tables  
* **Aggregation** of per-component metrics into a whole-mask summary, with optional additional focus on the *K* largest connected components  

---

## Installation

1. Clone this repository or download `VESSEL_METRICS.py`.  
2. Install dependencies (see `requirements.txt`).  

---

## Usage

```bash
python VESSEL_METRICS.py <input_path> 
    [--metrics METRIC [METRIC ...]] 
    [--topK NUMBER] 
    [--output_folder PATH] 
    [--no_conn_comp_masks] 
    [--no_segment_masks]

```

---

## Arguments

### Positional

- **`<input_path>`**  
  Path to the vessel mask file. Supported formats:  
  - NIfTI: `.nii`, `.nii.gz`  
  - NumPy: `.npy`, `.npz`  

### Optional

- **`--metrics METRIC [METRIC ...]`**  
  List of metrics to compute.  
  If not specified, **all metrics** are computed.  
  Valid options:  
  - `total_length`, `num_bifurcations`, `bifurcation_density`, `volume`  
  - `fractal_dimension`, `lacunarity`  
  - `geodesic_length`, `avg_diameter`  
  - `spline_mean_curvature`, `spline_mean_square_curvature`, `spline_rms_curvature`  
  - `num_loops`, `num_abnormal_degree_nodes`, `mean_loop_length`, `max_loop_length`  

- **`--topK INT`** _(default: None)_  
  Use only the **top-K largest connected components** (by total length) when computing aggregated summary metrics.  

- **`--output_folder PATH`** _(default: `./VESSEL_METRICS`)_  
  Directory to save results and intermediate files.  

- **`--no_conn_comp_masks`**  
  Disable saving connected-component-level masks (saves disk space).  

- **`--no_segment_masks`**  
  Disable saving per-segment masks (saves disk space and speeds up processing).  

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

* **total_length**: sum of all edge weights in the component.
* **num_bifurcations**: count of nodes with degree ≥ 3.
* **bifurcation_density**: `num_bifurcations / total_length`.
* **average_diameter**: length-weighted mean of the diameters along all edges in the connected component.  
  Each edge’s diameter is estimated as twice the average radius from the distance map at the two endpoints of the edge.  
* **volume**: approximates vessel volume by treating each edge as a cylinder:

  $\text{volume} = \sum_{(u,v)} \pi \left( \frac{r_u + r_v}{2} \right)^2 \cdot d_{uv},$

  where \$r\_u, r\_v\$ are radii from the distance map and \$d\_{uv}\$ is edge length.

### 4. Structural Metrics

* **num_loops**: number of independent cycles in the connected component (`len(nx.cycle_basis)`).
* **num_abnormal_degree_nodes**: nodes with degree greater than 3.
* **mean_loop_length**: average number of nodes in the cycles of the component.
* **max_loop_length**: maximum number of nodes in any cycle of the component.


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

  * **geodesic_length**: sum of consecutive node distances.
  * **avg_diameter**: average $2r$ along nodes.
  * **Spline tortuosity**:

    1. Fit a cubic B-spline (`splprep`) to segment points.
    2. Reparameterize by arc length for uniform sampling.
    3. Compute first and second derivatives w.r.t. arc length.
    4. Curvature: 
       $\kappa(s) = \frac{\lVert \mathbf{x}'(s) \times \mathbf{x}''(s) \rVert}{\lVert \mathbf{x}'(s) \rVert^3}$
    5. **Weighted node frequency**: points appearing multiple times are down-weighted  
       by their occurrence $n(s)$:
       \[
       \kappa_w(s) = \frac{\kappa(s)}{n(s)}, \quad
       \kappa_w^2(s) = \frac{\kappa(s)^2}{n(s)^2}
       \]
       
       This prevents overcounting in segments sharing nodes.
    6. Metrics:

       * **spline_arc_length**: $\int ds$
       * **spline_chord_length**: straight-line distance between endpoints
       * **arc_over_chord**: $\text{arc length} / \text{chord length}$
       * **spline_mean_curvature**: $\int \frac{\kappa(s)}{n(s)} \, ds$
       * **spline_mean_square_curvature**: $\int \frac{\kappa(s)^2}{n(s)^2} \, ds$
       * **spline_rms_curvature**: $\sqrt{\frac{1}{L} \int \frac{\kappa(s)^2}{n(s)^2} \, ds}$
       * **fit_rmse**: RMSE between spline and original points

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
```

