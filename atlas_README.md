

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
