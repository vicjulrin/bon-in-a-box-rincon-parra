---
title: "Study Area to grid"
output: 
  github_document:
    md_extension: +gfm_auto_identifiers
    preserve_yaml: true
    toc: true
    toc_depth: 6
---

Study Area to grid
================

- [Introduction](#introduction)
- [Set Session Parameters](#set-session-parameters)
- [Load libraries](#load-libraries)
- [Load data](#load-data)
- [Adjust grid projection](#adjust-grid-projection)
- [Create grid base](#create-grid-base)
- [Write results](#write-results)

## Introduction

This script converts a spatial study area layer into a spatial grid,
represented as a `.tif` raster file.

The script can be executed directly from R code or through the interface
of [studyarea_to_grid.R in the Bon in a Box
platform](studyarea_to_grid.R). This README serves as a guide for
running the script directly and understanding its functionality.

## Set Session Parameters

The session parameters for input and relative output paths of the script
are defined below as code. For more detailed information about each
parameter, refer to the accompanying YAML file
[studyarea_to_grid.yml](studyarea_to_grid.yml). This script is designed
to be run in a production environment within the [Bon in a Box
platform](https://github.com/GEO-BON/bon-in-a-box-pipelines), where
input and output parameters are provided through a JSON file.

This code uses the example input files included in the
[`input folder`](input) of this repository as a reference. However, it
should work for any input specified by the user. If it does not, please
report the issue to debug the code.

``` r
# Set session parameters ####

input<- list()
outputFolder<- "/output/StudyArea_to_grid"; dir.create(outputFolder, recursive = TRUE)

input$StudyArea_vector<- "/scripts/StudyArea_to_grid/input/Caldas.gpkg" # Text string with the path location for the spatial file containing the study area boundaries. Accepted file types include shapefile (.shp), GeoJSON (.json) or  GeoPackage (.gpkg).

input$EPSG_grid<- 4326 # "Numeric EPSG coordinate system in which the study grid will be projected. Must be a valid EPSG code that defines the coordinate reference system to ensure proper spatialization. EPSG website at https://epsg.org/home.html"

input$Resolution_grid<- 1000 # "Numerical value referring to the spatial resolution (pixel size) in square meters to create a grid of the study area."

input$name_layer<- "ID_grid" # "Text name of the raster layer to be created."
```

## Load libraries

The required libraries to execute the script are listed under
`required_packages`. The script checks if these libraries are already
installed on the system, and if any are missing, they are automatically
installed. Once all the required libraries are available, they are
loaded into the R environment, ensuring their functionalities are ready
for subsequent operations.

``` r
## Specify necessary libraries ####
required_packages <- c("magrittr", "this.path", "rjson", "data.table", "dplyr", "sf", "fasterize", "terra", "raster")
```

``` r
### Check and Install missing packages ####
installed <- installed.packages()[, "Package"]
missing_packages <- setdiff(required_packages, installed)
if (length(missing_packages)) {
  install.packages(missing_packages, repos = "https://packagemanager.posit.co/cran/__linux__/jammy/latest", dependencies = TRUE)
}

### Load libraries ####
lapply(required_packages, library, character.only = TRUE)
```

## Load data

The study area vector file is loaded and processed to ensure
compatibility with the desired coordinate reference system (CRS). The
CRS is defined as an object, `crs_basemap`, using the EPSG code provided
in the input parameters, establishing a consistent spatial framework.
The vector file `input$StudyArea_vector` is loaded and simplified into a
unified spatial representation, dissolving all internal boundaries into
a single feature. The aggregated vector is then reprojected to the
defined CRS, ensuring spatial alignment with the grid to be created.
Finally, the processed vector data is encapsulated in a `vector_polygon`
object of class `sf`, which will facilitate its subsequent analysis.

``` r
# Load data ####

#### Load crs projection ####
crs_basemap <- raster::crs(paste0("+init=epsg:", input$EPSG_grid))

#### Load vector StudyArea ####
vector_polygon <- terra::vect(input$StudyArea_vector) %>% 
  terra::aggregate() %>% 
  terra::project(crs_basemap) %>% 
  sf::st_as_sf()
```

## Adjust grid projection

To adjust the resolution in meters while ensuring alignment with the
coordinate system required by the user, a planar extent is generated
based on the `vector_polygon` object. This planar extent,
`ext_polygon_3395`, is defined in the EPSG 3395 coordinate system, which
represents a projected coordinate system suitable for accurate distance
and area calculations. Using this extent, a raster base is created with
the specified resolution and extent, ensuring that the grid framework is
consistent and accurate in the planar coordinate space. This raster,
`raster_base_3395`, serves as the logical base structure for accurately
defining the grid in meters and will later be transformed to conform to
the user’s specified coordinate system while retaining the expected
resolution.

``` r
## Adjust grid projection ####
# Adjusted projection to the planar coordinate system EPSG 3395.
ext_polygon <- sf::st_bbox(vector_polygon)
ext_polygon_3395 <- sf::st_bbox(vector_polygon) %>% 
  sf::st_as_sfc() %>% 
  sf::st_transform(3395) %>% 
  sf::st_as_sf() %>% 
  raster::extent()

raster_base_3395 <- raster::raster(ext_polygon_3395, crs = 3395, res = input$Resolution_grid)
```

## Create grid base

From the `raster_base_3395` object, a transformation is applied to match
the parameters of the `crs_basemap` object using, resulting in a
`raster_base` object, which represents the spatial grid framework
aligned to the user’s desired coordinate preferences. Using this grid
base, the `vector_polygon` object is rasterized to produce
`raster_study_area`, representing the study area in raster format. Each
cell or pixel of the `raster_study_area` is assigned a unique
identifier. Finally, the resulting layer is named according to the
user-defined `input$name_layer`, corresponding to the spatial study area
layer into a spatial grid, represented as a `.tif` raster file.

``` r
## Create grid base ####
raster_base <- raster_base_3395 %>% raster::projectRaster(crs = crs_basemap)

## Create Study Area base ####
raster_study_area <- fasterize::fasterize(vector_polygon, raster_base) %>% terra::rast()

StudyArea_grid <- raster_study_area %>% 
  terra::setValues(seq(ncell(.))) %>% 
  terra::mask(raster_study_area) %>% 
  setNames(input$name_layer)
```

## Write results

The final spatial grid, `StudyArea_grid`, is written to disk as a `.tif`
raster file to the folder path defined in the outputFolder parameter.

``` r
## Write results ####
StudyArea_grid_path <- file.path(outputFolder, paste0(input$name_layer, ".tif")) # Define the file path 
terra::writeRaster(
  StudyArea_grid, 
  StudyArea_grid_path, 
  gdal = c("COMPRESS=DEFLATE", "TFW=YES"), 
  filetype = "GTiff", 
  overwrite = TRUE
) # Write result

StudyArea_vector_path <- file.path(outputFolder, paste0("StudyArea_vector", "GeoJSON")) # Define the file path 
sf::st_write(vector_polygon, StudyArea_vector_path, delete_dsn = TRUE, driver = "GeoJSON")
```
