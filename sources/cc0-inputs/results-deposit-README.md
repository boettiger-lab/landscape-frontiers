# Output from: Landscape efficiency frontiers for biodiversity, climate mitigation and net economic value

This file describes the structure of the country-level outputs from [http://science.org/doi/10.1126/science.aea9058] http://science.org/doi/10.1126/science.aea9058).

See the published supplementary information (available at [http://science.org/doi/10.1126/science.aea9058 (http://science.org/doi/10.1126/science.aea9058)) for detailed methods, the inputs repository ([https://doi.org/10.5061/dryad.qjq2bvqw5](https://doi.org/10.5061/dryad.qjq2bvqw5)) for information about the inputs used to produce these results, and the code repository ([https://doi.org/10.5061/dryad.9s4mw6mxn](https://doi.org/10.5061/dryad.9s4mw6mxn)) for the code used in the analysis.

## Repository structure

The repository contains four single-country zips for the largest countries modeled (Brazil, Canada, China, USA), and 10 batches containing groups of smaller countries. The accompanying csv file `country_batches.csv` provides an index for which batch contains each country so that batches can be selectively downloaded. The file `lulc_codes.csv` describes the LULC code values used in the scenario maps.

## Country output structure

All countries have parallel output structures:

* **{date}.yml**
  * This is a config file that documents which global files were used to generate each of the country-level files. It provides a crosswalk between the global files in the associated data inputs repository and the files in the InputRasters folder.
* **InputRasters**
  * These rasters are subsetted from globally prepared input rasters to match the target country boundaries and are documented in the inputs repository ([https://doi.org/10.5061/dryad.qjq2bvqw5](https://doi.org/10.5061/dryad.qjq2bvqw5)). Please find the corresponding metadata in the README there.
* **ModelResults**
  * These rasters contain the spatial outputs of each of our environmental or economic models applied to the potential land use scenarios. They follow the naming convention `{scenario}_{variable}.tif`. The values for each raster are:
    * biodiversity: pixel level index incorporating species richness, endangered species presence, key biodiversity areas, endemic species, ecoregion rarity, and forest intactness. Units are pixel-level index score from 0-1. See SI section 3 for details.
    * carbon: Pixel level estimate of carbon stored in above and below ground biomass. Units are tons C/pixel. See SI section 2 for details.
    * cropland_value: Pixel-level estimate of cropland value given different management practices indicated by the scenario name. Units are USD 2015/pixel/year. See SI section 4 for details.
    * forestry_value: Pixel-level estimate of forestry value assuming sustainable management practices. Units are USD 2015/pixel/year. See SI section 6 for details.
    * grazing_methane: Pixel-level estimate of methane emitted from grazing livestock. Units are kg methane/pixel/year. See SI section 2 for details.
    * grazing_value: Pixel-level estimate of the economic value of grazing livestock. Units are USD 2015/pixel/year. See SI section 5 for details.
    * noxn_in_drinking_water: Deprecated. This model output was not used in the report and is not validated.
    * transition_cost: Pixel-level estimate of the annualized cost of converting from the original LULC to the scenario's LULC. Units are USD 2015/pixel/year. See SI section 7 for details.
* **national_boundary**
  * Defines the geographical extent modeled for the country.
* **OptimizationResults**
  * merged_summary_table.csv: Provides country-level summary statistics for the sustainable current baseline (first row with "sense"==0) and each of the optimization runs (rows with "sense"==1).
    These represent the aggregated values from the pixel-level ModelResults given the selection of SDU-level management decisions associated with the given optimization solution. Columns in this table are:
    * ID: an ID for that given solution, unique to a specific "sense".
    * sense: whether the row is for the baseline (sense=0) or for the optimization row (sense=1). A previous version of the optimization included minimizations (sense=-1), but this was not used in the final analysis.
    * net_econ_value, biodiversity, net_ghg_co2e, carbon, cropland_value, forestry_value, grazing_value, grazing_methane, production_value, transition_cost: country-level sums of corresponding values from the ValueTables (documented below).
    * noxn_in_drinking_water: Ignore.
    * econ: duplicate of net_econ_value.
    * *objective*_normed: transformed versions of the objective columns calculated as value divided by the maximum value in the column.
    * distance: distance between the baseline point and the optimized point measured as Euclidean distance in normed objective space considering net_econ_value, net_ghg_co2e, and biodiversity as the dimensions.
    * distance_urban: same as distance except we use biodiversity_normed_urban as the biodiversity objective axis for the distance calculation.
    * distance_econ: same as distance except we use biodiversity_normed_econ as the biodiversity objective axis for the distance calculation.
    * pareto_ecbw: is this a Pareto solution including economic, carbon, biodiversity, and water objectives. Not used.
    * pareto_ecb: is this a Pareto solution including economic, carbon, and biodiversity objectives. A Pareto solution equals or exceeds the baseline value on all dimensions.
    * pareto_dims: The number of dimensions along which this solution is a pareto solution (i.e., equals or exceeds the baseline value). Any value of 3 here will be TRUE in parcel_ecb.
  * FiguresAndMaps: contains a figure showing frontier plots with carbon and biodiversity plotted against net econ values, highlighting several key optimization solutions. These optimization solutions are:
    * nearest: the optimization solution with the smallest 'distance' score.
    * overall_max_biodiversity: the solution that maximizes the biodiversity index score.
    * overall_max_net_econ_value: the solution that maximizes net economic value (all agricultural uses minus transition costs).
    * overall_max_net_ghg_co2e: the solution that maximizes carbon storage.
    * pareto_max_biodiversity: the solution that maximizes the biodiversity index score among Pareto solutions (those that equal or improve on the baseline in all objective dimensions).
    * pareto_max_net_econ_value: the solution that maximizes net economic value (all agricultural uses minus transition costs) among Pareto solutions (those that equal or improve on the baseline in all objective dimensions).
    * pareto_max_biodiversity: the solution that maximizes carbon storage among Pareto solutions (those that equal or improve on the baseline in all objective dimensions).
    * Maps corresponding to the highlighted solutions are in the LU Maps folder, as either image files (pngs) or GeoTiffs (tifs)
* **ScenarioMaps**
  * Land use rasters used to define each of the potential management strategies. Land use codes as defined by the land use table documented in the inputs repository (included here for convenience). The scenarios that we modeled are described in detail in SI section 9. Across all scenarios, developed and water pixels are considered fixed and ineligible for conversion.
    Eligibility for cropland, grazing, and forestry expansion is determined based on biophysical and economic characteristics detailed in the SI. Briefly the included scenarios are:
    * all_econ: convert all eligible pixels to an economic use, preferring cropland > grazing > forestry with pixel-level characteristics determining where a given use is allowed. Used to calculate potential biodiversity score minimum.
    * all_urban: Convert all eligible pixels to urban. Used to calculate a potential biodiversity score minimum. Neither all-econ or all-urban was included in the landscape optimization choice set.
    * extensification_bmps_irrigated: expand cropland to all eligible pixels, assuming intensification to close yield gaps, use of irrigation, and use of best management practices.
    * extensification_bmps_rainfed: expand cropland to all eligible pixels, assuming intensification to close yield gaps, rainfed water management, and use of best management practices.
    * extensification_current_practices: expand cropland to all eligible pixels, assuming no change from current practices to close yield gaps or change from current water management.
    * extensification_intensified_irrigated: expand cropland to all eligible pixels, assuming intensification to close yield gaps, use of irrigation, and conventional management practices.
    * extensification_intensified_rainfed: expand cropland to all eligible pixels, assuming intensification to close yield gaps, rainfed water management, and conventional management practices.
    * fixed_area_bmps_irrigated: change management on current cropland pixels, assuming intensification to close yield gaps, use of irrigation, and use of best management practices.
    * fixed_area_bmps_rainfed: change management on current cropland pixels, assuming intensification to close yield gaps, rainfed water management, and use of best management practices.
    * fixed_area_intensified_irrigated: change management on current cropland pixels, assuming intensification to close yield gaps, use of irrigation, and conventional management practices.
    * fixed_area_intensified_rainfed: change management on current cropland pixels, assuming intensification to close yield gaps, rainfed water management, and conventional management practices.
    * forestry_expansion: expand sustainable forestry to all eligible pixels.
    * grazing_expansion: expand sustainable grazing to all eligible pixels.
    * restoration: convert all eligible pixels to natural cover based on predicted natural vegetation type.
    * sustainable_current: current (2015) land use updated to remove irrigation from areas where irrigation is projected to be unsustainable.
* **sdu_map**
  * Defines the geographical extent of the "Spatial Decision Units" (SDUs, ~8000 ha each) for the country, in vector and raster formats. The optimization routine, detailed in SI section 10, selects one management scenario per SDU, based on the summed objective values calculated from the ModelResults rasters aggregated in the ValueTables CSVs.
* **value_table_summary.csv**
  * Provides summary of country-level aggregated score for each service assuming full adoption of each scenario. Columns are as described in ValueTables/ModelResults, rows are the land use scenarios described in ScenarioMaps.
* **ValueTables**
  * Each value table gives the per-SDU values for adopting the named scenario for all modeled services. These tables are the direct inputs to the optimization, which will pick the optimal management to assign to each SDU based on preference weights across the main objectives (biodiversity, net_ghg_co2e, and net_econ_value). These are summed values from the rasters provided in ModelResults (i.e. zonal stats using the sdu_map as the aggregating geometry), with the exception of production_value, net_econ_value, and net_ghg_co2e, which are calculated from the other summarized results. Column names and values match the descriptions given above (see ModelResults section), with the modification that all units are per SDU rather than per pixel.
    For the added columns, we have:
    * biodiversity: SDU-level index incorporating species richness, endangered species presence, key biodiversity areas, endemic species, ecoregion \___, and forest intactness. Units are index score from 0-1. See SI section 3 for details.
    * carbon: SDU-level estimate of carbon stored in above and below ground biomass. Units are tons C/SDU. See SI section 2 for details.
    * cropland_value: SDU-level estimate of cropland value given different management practices indicated by the scenario name. Units are USD 2015/SDU/year. See SI section 4 for details.
    * forestry_value: SDU-level estimate of forestry value assuming sustainable management practices. Units are USD 2015/SDU/year. See SI section 6 for details.
    * grazing_methane: SDU-level estimate of methane emitted from grazing livestock. Units are kg methane/SDU/year. See SI section 2 for details.
    * grazing_value: SDU-level estimate of the economic value of grazing livestock. Units are USD 2015/SDU/year. See SI section 5 for details.
    * noxn_in_drinking_water: Deprecated. This model output was not used in the report and is not validated.
    * transition_cost: SDU-level estimate of the annualized cost of converting from the original LULC to the scenario's LULC. Units are USD 2015/SDU/year. See SI section 7 for details.
    * production_value: sum of cropland_value, forestry_value, and grazing_value. Units are USD 2015/SDU/year.
    * net_econ_value: Calculate as production_value - transition_cost. Units are USD 2015/SDU/year.
    * net_ghg_co2e: Combines carbon storage and methane emissions to a common measure in tons CO2e/SDU/year. Methods in SI section 2.

## Code/Software

The code used to produce these outputs is available in a separate repository
([https://doi.org/10.5061/dryad.9s4mw6mxn](https://doi.org/10.5061/dryad.9s4mw6mxn)).