# Project Vaayu

The official repository for Project Vaayu .


## Background

1. The Honorable Prime Minister launched the SVAMITVA Scheme on National Panchayati Raj Day, 24th April 2020, with the resolve to enable the economic progress of Rural India by providing "Record of Rights" to every rural household owner. The scheme aims to demarcate inhabited (Abadi) land in rural areas through the latest surveying drone technology, Continuous Operating Reference System (CORS), and Geographic Information System (GIS) technology. The scheme covers multiple aspects such as facilitating the monetization of properties and enabling bank loans, reducing property-related disputes, and supporting comprehensive village-level planning.

2. With the use of the latest drone technology and CORS for the Abadi land survey, the high-resolution and accurate image-based maps (with 50 cm resolution) have facilitated the creation of the most durable record of property holdings in areas without legacy revenue records. These accurate image-based maps provide a clear demarcation of land holdings in a very short time compared to on-ground physical measurement and mapping of land parcels.

## Description of the Problem Statement

i. **Develop an AI model capable of identifying key features in orthophotos with high precision:**  
   Use AI/ML techniques for the extraction of the following features from SVAMITVA Drone Imagery using a cloud-based solution:  
   - Building footprint extraction (built-up area from the drone image and classified rooftop based on observation in the imagery as RCC, Tiled, Tin, and Others). These built-up areas can be used for various services such as solar energy calculation, property tax calculation, etc.  
   - Road feature extraction  
   - Waterbody extraction, etc.

ii. **Achieve a target accuracy of 95% in feature identification.**

iii. **Optimize the model for efficient processing and deployment.**

## Navigating this Repository:

## The Data

### File Formats

The first issue we had to tackle when solving this problem was understanding the data given to us. The Ministry of Panchayati Raj provided us a dataset of geospatial data that included .ecw base-maps and .shp vector files. ECW stands for Enhanced Compression Wavelet and is a proprietary file format used for storing high-resolution geographical images. SHP stands for ShapeFiles which is a vector data format that stores geographic information, such as the location, shape, and attributes of features.

Read more about ECW files [here.](https://en.wikipedia.org/wiki/ECW_(file_format))
Read more about Shapefiles [here.](https://www.precisely.com/glossary/shapefile)




### Dealing with the Data

Since this was our first time dealing with geospatial data, these file formats were completely foreign for us. It took us a while to learn QGIS and explore the data. After exploring the data given to us for a while, it became apparent that the ECW files were meant to be the expected "input" data for our AI model, whereas the SHP files were manually created annotations that were meant to be the expected output of the ML model.

The first problem with the ECW files was that they were very high-resolution. Also, you cannot directly process ECW files in GDAL unless you have a license (you can only read the files for free). So we would need to convert the data from ECW to something else.

We first converted the images from ECW to TIF file formats, which resulted in us obtaining very large TIF files (multiple gigabytes large). We could not process these files directly with segmentation models. So we decided to go for a sliding-window approach where the image would be split up into tiles and each tile would be processed independently. This process would be followed during training and inference. We went for a resolution of 3000x3000 for the tiles.


#### GDAL command used for ECW to TIF Conversion:
```
gdal_translate -of GTiff -co "COMPRESS=LZW" input.ecw output.tif
```

#### Tiling

The same process was followed for the Shapefiles as well. First, the Shapefile was rasterized, after which some processing was applied to make the output data into binary masks that would be suitable for training an ML model. Basically, the masks had black backgrounds and white masks for the foreground elements that we wanted to segment out. Then this large rasterized TIF file was also tiled to get "output" data. The masks also had a resolution of 3000x3000.



#### Shapefile rasterization settings we used in QGIS:
```
{
  "area_units": "m2",
  "distance_units": "meters",
  "ellipsoid": "EPSG:7030",
  "inputs": {
    "BURN": 1.0,
    "DATA_TYPE": 0,
    "EXTENT": "Calculate from Layer and Choose your ECW Raster Layer",
    "EXTRA": "",
    "FIELD": null,
    "HEIGHT": {pixel height of base raster layer},
    "INIT": null,
    "INPUT": "input_shapefile.shp",
    "INVERT": false,
    "NODATA": 0.0,
    "OPTIONS": "COMPRESS=DEFLATE|PREDICTOR=2|ZLEVEL=9|mask=1|ALPHA=YES|TILED=YES",
    "OUTPUT": "output_raster.tif",
    "UNITS": 1,
    "USE_Z": false,
    "WIDTH": {pixel height of base raster layer}
  }
}
```

The above settings are very important - if you don't set these properly, you can end up generating rasterized shapefiles that are hundreds of gigabytes large. (💀) I speak from experience here.

#### GDAL commands for post-processing the rasterized Shapefile

```
gdal_rasterize -l layer_name -burn 1.0 -tr 1.0 1.0 -te [SET EXTENT OF UNDERLYING ECW LAYER. Use "Calculate from Layer" option in QGIS.] -a_nodata 0.0 -ot Byte -of GTiff -co COMPRESS=DEFLATE -co PREDICTOR=2 -co ZLEVEL=9 -co TILED=YES "input.shp" temp_raster.tif

gdal_translate -of GTiff -b 1 -mask 1 -a_nodata 0 -co ALPHA=YES temp_raster.tif raster_with_alpha.tif

gdal_calc -A raster_with_alpha.tif --outfile=final_raster_white.tif --calc="255*(A>0)" --NoDataValue=0 --type=Byte
```

## The Models

We explored a lot of models during the duration of the hackathon. Here are the results from a few:

### Meta's Segment Anything Model (SAM)
This was the first model we tried, and we tested it out on SAM's official website - using the automatic mask generator.


#### LangSAM
We also attempted to use LangSAM which applied a text encoder to SAM and allowed you to use input text prompts for image segmentation. This implementation was fine for input images where the buildings were few and far between. 

For example, 


But when the buildings were closer together, LangSAM struggled to keep up due to overlapping bounding boxes.



#### Verdict:
SAM was performing well at making precise masks but it was also selecting a lot of random surrounding objects. The base model was also not able to perform well for roads and water bodies.

### Detectron2
We found a few posts online suggesting that Detectron2 was better than SAM for image segmentation. So we decided to proceed with fine-tuning Detectron2. For this, we had to create a COCO dataset from the training data we had.

### FPN - Feature Pyramid Networks

We decided to try fine-tuning an FPN model.


#### Verdict:
FPNs only performed well for roads. When it came to water bodies, due to the lack of training data, it was not able to learn enough from the data. For buildings, it seemed to face the same problem that Detectron2 faced - it was generating very blotchy outputs. This was a general problem we faced with all of our segmentation models and architectures that we tried. Even UNet, the architecture that ended up winning the competition, showed the same issues in our early testing.

## Aaand finally, the winner is....
UNet++. It was always UNet! Seeing UNet get the best performance on this task and also seeing it win the entire competition was like a back-to-basics moment for me. Sometimes, the simplest solution is the correct solution. 

UNet++ (U-Net with Nested Skip Pathways) is an advanced architecture for image segmentation tasks, primarily used in medical imaging and computer vision. It builds upon the original U-Net model by enhancing its skip connections through nested pathways. This structure allows the network to capture more fine-grained features at various resolutions, improving segmentation accuracy. The key innovation of UNet++ is the introduction of dense skip pathways and deep supervision, which helps refine the features learned at different levels and improves the overall performance of the model. This makes UNet++ particularly effective for complex segmentation tasks where precise details are critical.


## The Solution (Inference Pipeline)
At the competition, we prepared a pipeline that used FPNs for roads and buildings and Detectron2 for water bodies. FPN for roads and Detectron2 for water bodies showed great results but the performance for FPN for buildings was not acceptable - before we could try anything else for buildings, our time at the hackathon ran out. However, the updated version of the solution will solve the issue with buildings.


Steps in the pipeline:

1. The user uploads an ECW file.
2. The ECW file is converted to a large TIF file using GDAL Translate.
3. The large TIF file is converted to a grid of smaller tiles - each 3000x3000 pixels. This is done using the any_to_tif_grid.bat script.
4. Each tile is passed through different models. Specifically, each image is passed through one instance of each of the following models - the road segmentation model, the water body segmentation model, the rooftop type segmentation model (one model each for RCC, Tiled, Tin and Other rooftops).
5. Each inference will produce one output tile - which is saved in a corresponding output folder for each feature.
6. All the output tiles are then recombined into one large raster file using GDAL Warp.
7. The raster file is then converted to a ShapeFile using GDAL Polygonize.

One issue here is that when the input tiles pass through the model, the output TIF tiles do not contain their Geographical Information anymore - i.e. they go from GeoTiffs to normal TIF files. Due to this, they do not get polygonized by GDAL Polygonize. To solve this, we can re-apply the geographical information present in the input tiles to the output tiles - since they correspond with each other 1:1 and thus, have the same geographical information. You can use the following gdalwarp command to copy the geo-referencing information from the GeoTIFF tile to the normal TIFF output mask tile: (you will have to run this command for each tile - I suggest creating a batch script)

```
gdalwarp -s_srs EPSG:4326 -t_srs EPSG:4326 -of GTiff -co "TILED=YES" -co "COMPRESS=LZW" input_normal.tif input_georeferenced.tif
```

GDAL Warp is used to merge the generated output TIF grid back to one large TIF file as follows:

```
gdalwarp -of GTiff path\to\grid_tifs\*.tif path\to\output_combined.tif
```

Finally, using GDAL Polygonize, the recombined TIF file would be polygonized to a Shapefile which was the desired output filetype:

```cmd
gdal_polygonize built_up_raster_lalpur.tif -f "ESRI Shapefile" test_builtup_lalpur.shp
```

The solution also expected us to design a deployment pipeline for the AI model that we created using AWS, ReST APIs and a microservices architecture.

