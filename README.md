# GA4GH Environmental Exposure Linkage Metadata (draft 0.1.0)

Environmental exposures such as air pollution, diet, physical activity, and social stress can influence the onset of disease. These effects often accumulate slowly over time and can depend on multiple factors such as lifestyle, occupation, and living conditions. Currently, environmental data is captured in different ways by research groups internationally, making it hard to compare or combine this information. We have set up the Human Exposome.

The Human Exposome Data Standards Study Group is working to solve a big problem in global health research: there are not yet consistent, shared ways to describe how the environment affects human health — especially over long periods of time and across different populations. Find out more here: https://doi.org/10.5281/zenodo.21877291

We are creating a machine-readable *Place-Based Exposures Metadata Schema*. Each
**record** describes one exposure variable linked to one cohort. Find out more about the work in 

## Repository layout

```
schema/exposure_linkage_schema.yaml   # LinkML source - edit this file only
schema/exposure_linkage.schema.json   # generated JSON Schema 2020-12 - do not hand-edit
tools/linkml_to_jsonschema.py         # generator (stand-in for LinkML's gen-json-schema)
tools/validate.py                     # validate metadata files; exit 1 on any error
examples/valid/*.yaml                 # must pass CI
examples/invalid/*.yaml               # must fail CI (regression tests for the rules)
.github/workflows/validate.yml
```

## Validating a submission

```bash
pip install jsonschema pyyaml
python tools/validate.py my_cohort_metadata.yaml
```

YAML pitfall: quote country codes in data files (`["NO"]`, not `[NO]`). Unquoted `NO`
(Norway) is read as boolean `false` by YAML 1.1 parsers. Submitting JSON avoids this.

## What changed from the spreadsheet

| CSV field | Machine-readable form | Why |
|---|---|---|
| Cohort / Study Name | `cohort.name` + `cohort.acronym` | No parsing of "Name (ACR)" |
| Country of data collection | `cohort.collection_countries[]` (ISO 3166-1 enum) | Rejects invalid codes such as `UK` |
| Exposure Domain | `exposure.domain` enum + `domain_other` | "Other (specify)" becomes a required field |
| Exposure Variable Name | `exposure.name` | |
| Ontology / Taxonomy Term ID | `exposure.ontology_terms[]` as CURIEs (`INCHIKEY:`, `PUBCHEM:`, `ENVO:`…) | Resolvable identifiers; empty list instead of "N/A" |
| Origin of the data | `source_data.origin` enum | Added `REANALYSIS` (e.g. ERA5) |
| Model / Dataset Name and Version | `dataset_name` + `dataset_version` | Separate fields |
| Data Source Citation / URL | `citations[]` with `doi`, `url`, `accessed`, `text`, `used_for_linkage` | Replaces semicolon lists and the `*` marker |
| Geographic Coverage | `geographic_coverage.scope` + `regions[]` (ISO 3166-1/-2) + `description` | `EU27` and `GLOBAL` as explicit scopes |
| Spatial Resolution | `spatial_resolution.type` + `cell_width`/`cell_height` or `admin_unit_*` | Numbers instead of "100 m × 100 m" strings |
| Date Coverage – Start / End | `temporal_coverage.start`/`end` (ISO 8601 partial dates) + `ongoing` + `last_updated` | "Ongoing" becomes a boolean, not a magic string |
| Temporal Resolution | enum + `update_frequency` (ISO 8601 duration, e.g. `P5Y`) | |
| Validation Status | `validation_metrics[]`: `{metric, validation_design, value, unit}` | Metrics can be queried and compared across cohorts |
| Uncertainty | `uncertainty`: `{type, value, lower, upper, unit, carried_to_participant_level}` | |
| Unit of Measure | UCUM code (`ug/m3`, `[ppb]`, `Cel`…) | Machine-convertible units |
| Spatial Linkage Unit | enum + `centroid_weighting` + `centroid_areal_unit` + `population_dataset` | "(specify type)" becomes conditional required fields |
| Spatial Linkage Operation | enum + `buffer_radii_m[]`, `network_*`, `kernel_*`, `intersection_min_overlap_pct` | |
| Geocoding Precision | enum + `geocoding_mixed_breakdown` | |
| Summary Statistic / Aggregation | enum + `aggregation_n` | Buffer geometry is recorded once, under linkage |
| Buffer Distance | merged into `buffer_radii_m[]` (numeric metres) | Removes duplication |
| Missing Data / Coverage | `participant_coverage_pct` (0–100) + `coverage_unit` + `study_area_coverage_pct` | |
| Exposure Window | enum + `exposure_window_years` + `exposure_window_detail` | |
| Residential History | enum + `residential_history_detail` | |
| Linkage Date | `linkage_date` (ISO date) | |
| Geocoding Database Version | `geocoder` + `relinkages[]` `{date, reason}` | |
| Creator(s) | `creators[]`: `{name, email, orcid, role}` | ORCID gives a persistent identity |

**Tiers.** CORE fields are `required: true`. Recommended fields carry LinkML's
`recommended: true`; LinkML tooling reports them as warnings, and JSON Schema ignores them.

## Conditional rules enforced

| If… | …then required |
|---|---|
| `domain = OTHER` | `domain_other` |
| `operation = CIRCULAR_BUFFER` | `buffer_radii_m` |
| `operation = NETWORK_BUFFER` | `network_mode` |
| `operation = KERNEL_DENSITY_WEIGHTED` | `kernel_bandwidth_m` |
| `aggregation` is `MULTI_YEAR_MEAN` or `PERCENTILE` | `aggregation_n` |
| `spatial_linkage_unit` is any `*_CENTROID` | `centroid_weighting` |
| `centroid_weighting = POPULATION_WEIGHTED` | `population_dataset` |
| `geocoding_precision = MIXED` | `geocoding_mixed_breakdown` |
| spatial resolution is `GRID_METRIC` or `GRID_DEGREE` | `cell_width`, `cell_height` |
| spatial resolution is `ADMINISTRATIVE_UNIT` | `admin_unit_name` |
| `scope = COUNTRIES_OR_REGIONS` | `regions` |
| `temporal_resolution = PERIODIC_UPDATE` | `update_frequency` |
| `ongoing = true` | `last_updated` |
| any citation | at least one of `doi`, `url`, `text` |

## Open items for the Work Stream

- Bind enum values to ontology terms (ENVO, ExO, NCIT) using LinkML `meaning:`. They are
  deliberately left unbound so that no identifiers are guessed.
- Decide whether to reuse GA4GH GKS / Phenopackets building blocks, such as an
  `OntologyClass` structure for `ontology_terms`.
- Register a persistent `w3id.org` namespace to replace the placeholder `id`.
- Confirm UCUM handling for dB(A) and other weighted noise metrics, which have no UCUM code.
