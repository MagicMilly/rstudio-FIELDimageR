
# rstudio-FIELDimageR

FIELDimageR in a RStudio container with `verse` dependencies, based on [Vice RStudio Docker container](https://hub.docker.com/r/cyversevice/rstudio-verse) for CyVerse VICE.

# Docker Instructions

## Run this Docker locally or on a Virtual Machine

To run these containers, you must first pull them from DockerHub.

```
docker pull agdrone/cyverse_fieldimager:latest
```

```
docker run --rm -v /$HOME:/app --workdir /app -p 8787:80 -e REDIRECT_URL=http://localhost:8787 agdrone/cyverse_fieldimager:latest
```

The default username is `rstudio` and password is `rstudio1`.
To reset the password, add the flag `-e PASSWORD=<yourpassword>` in the `docker run` statement (replacing <yourpassword> with the actual password).

## Build your own Docker container and deploy on CyVerse VICE

This container is intended to run on the CyVerse data science workbench, called [VICE](https://cyverse-visual-interactive-computing-environment.readthedocs-hosted.com/en/latest/index.html). 

###### Developer notes

To build your own container with a Dockerfile and additional dependencies, pull the pre-built image from DockerHub:

```
FROM agdrone/cyverse_fieldimager:latest
```

Follow the instructions in the [VICE manual for integrating your own tools and apps](https://cyverse-visual-interactive-computing-environment.readthedocs-hosted.com/en/latest/developer_guide/building.html).

# Running the VICE app
If running the docker image on VICE, the default username is `rstudio` and password is `rstudio1`.

Additional information is available in the CyVerse [RStudio Quick Start](https://learning.cyverse.org/projects/vice/en/latest/user_guide/quick-rstudio.html) documentation. 

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

We are using an example source image named `EX1_RGB.tif`.
You can substitute the name of any georeferenced image file you'd like.

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

```R
latlon_crs<-crs('+proj=longlat +datum=WGS84 +no_defs')
latlon_shape<-spTransform(EX1.Shape$fieldShape, latlon_crs)

multi_polys<-attr(latlon_shape, 'polygons')
json<-'{"type": "FeatureCollection", "name": "FIELDimageR", "features": ['
npolys<-length(multi_polys)
separator<-''
for (i in 1:npolys) {
  poly<-multi_polys[[i]]
  polys<-attr(poly, 'Polygons')
  coords<-coordinates(polys[[1]])
  ncoords<-dim(coords)[[1]]
  json<-paste(json, separator, '{"type": "Feature", "properties": { "id": "', c(i), '", "observationUnitName":"', c(i), '"}, "geometry": { "type": "Polygon", "coordinates": [ [ ')
  coord_separator<-''
  for (c in 1:ncoords) {
    json<-paste(json, coord_separator, "[", coords[c, 1], ",", coords[c, 2], "]")
    coord_separator<-', '
  }
  json<-paste(json, "] ] } }")
  separator<-', '
}
json<-paste(json, ']}')


fileConn<-file("plots.json")
writeLines(json,fileConn)
close(fileConn)
```