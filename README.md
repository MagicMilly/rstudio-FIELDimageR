
# rstudio-FIELDimageR

FIELDimageR in a RStudio container with `VICE` dependencies, based on [Vice RStudio Docker container](https://hub.docker.com/r/cyversevice/rstudio-verse) for CyVerse VICE.

## Overview
The original documentation for [FIELDimageR](https://github.com/OpenDroneMap/FIELDimageR) provides information on its capabilities.

This documentation is for writing [GeoJSON](https://datatracker.ietf.org/doc/rfc7946/?include_text=1) plot files by leveraging the functionality inherent in FIELDimageR.
If you have access to an existing VICE app you can skip ahead to that [section](#vice);

There is a non-CyVerse Docker image available named [agdrone/fieldimager](https://hub.docker.com/repository/docker/agdrone/fieldimager).
Refer to the [rocker/rstudio](https://hub.docker.com/r/rocker/rstudio) instructions for running this Docker image.
The instructions starting with the [preparation step](#preparation) onward can be used with this image.

# Quick Start
Use the following steps to quickly get started with the CyVerse app:
1. Log into [CyVerse](https://de.cyverse.org/de/) Discovery Environment
2. Click on the Apps tile to open the `Apps` window
3. If needed, find the FIELDimageR app
4. Click on the name of the app to open the FIELDimageR analysis launch window
5. Specify a folder containing the source image(s) in the `Sources` section of the analysis launch window
6. Click the `Launch Analysis` button to start the app
7. Wait for a CyVerse notification with the text: `access your running analysis here`
8. Click the link and wait until the CyVerse login screen appears
9. Log into CyVerse with your credentials
10. Log into the RStudio instance using the username `rstudio` and password `rstudio1` credentials
11. Follow the instructions in the [preparation step](#preparation) to generate the plot GeoJSON file

# Docker Instructions

## Run this Docker locally or on a Virtual Machine

To run these containers, you must first pull them from DockerHub.

```
docker pull agdrone/cyverse_fieldimager:latest
```

```
docker run --rm -v /$HOME:/app --workdir /app -p 8787:80 -e REDIRECT_URL=http://localhost:8787 agdrone/cyverse_fieldimager:latest
```

#### Local Credentials
The default username is `rstudio` and password is `rstudio1`.
To reset the password, add the flag `-e PASSWORD=<yourpassword>` in the `docker run` statement (replacing \<yourpassword\> with the actual password).

## Build your own Docker container and deploy on CyVerse VICE
A pre-built container is available on [DockerHub](https://hub.docker.com/repository/docker/agdrone/cyverse_fieldimager).

The built container is intended to run on the CyVerse data science workbench, called [VICE](https://cyverse-visual-interactive-computing-environment.readthedocs-hosted.com/en/latest/index.html). 

##### Developer notes

To build your own container with a Dockerfile and additional dependencies, get the [Dockerfile](https://github.com/Chris-Schnaufer/rstudio-FIELDimageR) from the GitHub repository.

Next, [build](https://docs.docker.com/engine/reference/commandline/build/) the Docker image using a command line prompt.

A sample command line is shown next; you will need to replace parameters for your environment.
```bash
docker build -t agdrone/cyverse_fieldimager:1.0 -f 1.0.0/Dockerfile ./
```
The following parameters are:
* `docker` this is the Docker command to run
* `build` instructions docker to build an image
* `-t` indicates that the tag (name) of the image follows
* `agdrone/cyverse_fieldimager:1.0` the tag of the built image; this name should be changed for your environment
* `-f` indicates that the path of the Dockerfile to use follows
* `1.0.0/Dockerfile` the relative path to the Dockerfile to use; the path to the Dockerfile may be different on your system
* `./` indicates that the current folder is to be used as needed

Once the Docker image is built, [docker push](https://docs.docker.com/engine/reference/commandline/push/) can be used to upload the image to where CyVerse VICE can access it.
The following command uploads the image built in the previous step to [DockerHub](https://hub.docker.com/).
You will need to replace the image tag in the following command with the one specified previously.
```bash
docker push agdrone/cyverse_fieldimager:1.0
```

Follow the instructions in the [VICE manual for integrating your own tools and apps](https://cyverse-visual-interactive-computing-environment.readthedocs-hosted.com/en/latest/developer_guide/building.html).

# Running the VICE app <a name="vice"/>
Additional information is available in the CyVerse [RStudio Quick Start](https://learning.cyverse.org/projects/vice/en/latest/user_guide/quick-rstudio.html) documentation. 

#### VICE Credentials
If running the docker image on VICE, the default username is `rstudio` and password is `rstudio1`.

# Generating Plot GeoJSON File
This document describes how to use [FIELDimageR](https://github.com/OpenDroneMap/FIELDimageR) to generate a Plot boundary [GeoJSON](https://datatracker.ietf.org/doc/rfc7946/?include_text=1) file.

These instructions assume that you have access to a running Docker container that already has the requirements for FIELDimageR installed.
Refer to the [installation](https://github.com/OpenDroneMap/FIELDimageR#Instal) instructions if needed.

We will only be referencing the minimum steps needed for generating the GeoJSON file.
There is additional functionality available with FIELDimageR.

## Preparation Steps <a name="preparation" />
This section lists the steps needed to define the plot boundaries ([saving the plot geometries](#generate) follows).

<hr />

The steps used here are a non-consecutive subset of those available with FIELDimageR with additional commands added.

- [2. Select the targeted field from the original image](https://github.com/OpenDroneMap/FIELDimageR#2-selecting-the-targeted-field-from-the-original-image)
- [3. Rotating the image](https://github.com/OpenDroneMap/FIELDimageR#3-rotating-the-image)
- [5. Building the plot shape file](https://github.com/OpenDroneMap/FIELDimageR#5-building-the-plot-shape-file)

<hr />

We are using an example source image named [EX1_RGB.tif](https://drive.google.com/file/d/1S9MyX12De94swjtDuEXMZKhIIHbXkXKt/view).
You can substitute the name of any georeferenced image file you'd like.

**Specify libraries needed**

Load the needed libraries
```R
library(FIELDimageR)
library(raster)
library(geojsonio)
library(stringi)
```

**Load the image**

Load the source image with the following command.
```R
EX1<-stack("EX1_RGB.tif")
```

**Crop to field of interest**

Run the following step to crop the image to the field.
This will make it easier to specify the plot boundaries.
```R
EX1.Crop <- fieldCrop(mosaic = EX1)
```

**Rotate the image**

The cropped image needs to be rotated so that the columns are vertically aligned (facing  "up).
The `clockwise` parameter is shown as having a `F` (False) value indicating counter-clockwise rotation.
If you need clockwise rotation, or your attempts to vertically align the columns isn't working out, try setting the parameter to `T` (True).
```R
EX1.Rotated<-fieldRotate(mosaic = EX1.Crop, clockwise = F)
```

**Restore Geographic information**

The rotation may have changed the geographic information for the image.
The following commands ensures it's correct.
```R
crs(EX1.Rotated)<-crs(EX1.Crop)
extent(EX1.Rotated)<-extent(EX1.Crop)
```

**Defining the plot boundaries**

Now that the image is prepared the plot boundaries can be defined.

Be sure to specify the correct count of columns (`ncols`) and rows (`nrows`) for your field.

```R
EX1.Shape<-fieldShape(mosaic = EX1.Rotated, ncols = 14, nrows = 9)
```

## Generate GeoJSON file <a name="generate" />
The following code takes the plot boundaries produced in the [Preparation](#preparation) section above and writes the GeoJSON to a file named 'plots.json'.

Ensure coordinate system of polygons is in the expected Lat-Long WGS84 (EPSG 4326).
```R
latlon_crs<-crs('+proj=longlat +datum=WGS84 +no_defs')
latlon_shape<-spTransform(EX1.Shape$fieldShape, latlon_crs)
```

Generate the JSON to write:
```R
multi_polys <- attr(latlon_shape, 'polygons')
json <- '{"type": "FeatureCollection", "name": "FIELDimageR", "features": ['
npolys <- length(multi_polys)

separator<-''
for (i in 1:npolys) {
  poly <- multi_polys[[i]]
  polys <- attr(poly, 'Polygons')
  coords <- coordinates(polys[[1]])
  ncoords <- dim(coords)[[1]]
  json <- paste(json, separator, 
               '{"type": "Feature",
                "properties": { "id": "', c(i), '",
                "observationUnitName":"', c(i), '"}, 
                "geometry": { "type": "Polygon",
                "coordinates": [ [ ')
  coord_separator <- ''
  for (c in 1:ncoords) {
    json<-paste(json, coord_separator, "[", coords[c, 1], ",", coords[c, 2], "]")
    coord_separator <- ', '
  }
  json<-paste(json, "] ] } }")
  separator <- ', '
}
json<-paste(json, ']}')
```

Write JSON to the file.
```R
fileConn<-file("plots.json")
writeLines(json, fileConn)
close(fileConn)
```
