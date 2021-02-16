# GDAL-twms-with-NASA-GIBS
How to make use of the NASA/GIBS services using GDAL and the tWMS client

## Introduction
[GIBS](https://earthdata.nasa.gov/eosdis/science-system-description/eosdis-components/gibs) is a component of the NASA Earth Observation System Data and Information System (EOSDIS) which provides online access to current and historical imagery derived from multiple satellite sources. The core web service is tiled, which means that response is fast and that it is not necessary to download a complete dataset if the area of interest is only a small region.  
[GDAL](https://gdal.org/) is a well known geospatial data access library which is part of most GIS applications. GDAL can read (and sometimes write) geospatial raster and vector data in many formats, the format specific operations being handled by a GDAL driver.  

One of these drivers is [GDAL WMS](https://gdal.org/drivers/raster/wms.html), sharing a name with one of the oldest and best known [OGC standards](https://www.ogc.org/standards/wms), a non-tiled or dynamic map service. As expected, the GDAL WMS driver can be used to connects to a WMS service, but can also connect to many other raster map services, so the name is a little bit misleading. The WMS driver can access many of the known tiled and non-tiled protocols. In turn, the WMS driver is used by many other GDAL drivers, for example it is used by the GDAL [WMTS driver](https://gdal.org/drivers/raster/wmts.html).
GDAL WMS has a common part that handles interfacing with GDAL, making the requests and receiving the responses, decoding the received data as needed and a few other things. Since using tiles and multiple resolutions (known as a pyramid) is well supported in GDAL and widespread in the GIS domain, the core of the WMS driver is based on this model, and tile requests are made even when accessing a dynamic service that could provide it's own subsetting and resampling. The WMS driver uses an XML configuration to select the server, protocol and any other parameters, as detailed here: [GDAL WMS](https://gdal.org/drivers/raster/wms.html). For most minidrivers there are lots of options that have to be just right and matching the web service to make it work. A GDAL WMS MetaDriver exists to help with the task of generating such XML configuration files.

One of these WMS-minidrivers is TiledWMS (tWMS), which is a tiled extension for the OGC WMS. The tWMS minidriver lets the server provide the information needed to connect, greatly simplifying the end user task of building the WMS file. tiledWMS is one of the offered [GIBS APIs](https://wiki.earthdata.nasa.gov/display/GIBS/GIBS+API+for+Developers#GIBSAPIforDevelopers-TiledWebMapService(TWMS)), and as such can be used to simplify access to the GIBS data. The rest of this document will mostly use the GDAL command line tools such as [gdalinfo](https://gdal.org/programs/gdalinfo.html) and [gdal_translate](https://gdal.org/programs/gdal_translate.html) to simplify the access to the tiledWMS driver. However, once a valid tiledWMS XML configuration file is generated, the dataset it represents can be used by any application that uses GDAL.

## List of available datasets

TiledWMS adds a *GetTileService* call to the OGC WMS, the XML document returned is what contains all the information needed to configure and connect to any dataset. As documented by the GIBS API page, the GIBS server URL for this request is [https://gibs.earthdata.nasa.gov/twms/epsg4326/best/twms.cgi?request=GetTileService](https://gibs.earthdata.nasa.gov/twms/epsg4326/best/twms.cgi?request=GetTileService)  
This URL can be used by gdalinfo to get a list of available datasets:
```
gdalinfo "https://gibs.earthdata.nasa.gov/twms/epsg4326/best/twms.cgi?request=GetTileService"
```
It takes a few seconds and a lot of output will be generated. This is the output from the WMSMetaDriver, which recognized the response and parsed it. The dataset itself is just a placeholder, the real output is in the metadata, which exposes all the datasets available on the server as GDAL subdatasets. There are close to one thousand different datasets available on that server. Each one will get a *SUBDATASET_\<N>_NAME* and a *SUBDATASET_\<N>_DESC*, where \<N> is the subdataset number. The SUBDATASET_NAME is picked right out of the GetTileService response and should be a readable, informative description of the respective dataset. Let's pick subdataset 479 and take a closer look:
```
gdalinfo "https://gibs.earthdata.nasa.gov/twms/epsg4326/best/twms.cgi?request=GetTileService" |grep 479
```
The output will be:
```
  SUBDATASET_479_NAME=<GDAL_WMS><Service name="TiledWMS"><ServerUrl>https://gibs.earthdata.nasa.gov/twms/epsg4326/best/twms.cgi?</ServerUrl><TiledGroupName>MODIS Terra CorrectedReflectance TrueColor tileset</TiledGroupName></Service></GDAL_WMS>
  SUBDATASET_479_DESC=Corrected Reflectance (True Color, MODIS, Terra)
 ```
The description makes sense. This specific dataset is the global Terra MODIS True Color mosaic. The SUBDATASET_479_NAME looks like XML. That is exactly the XML needed by the tiledWMS driver to connect to the respective dataset. Let's use gdalinfo to get more details:
```
gdalinfo -sd 479 "https://gibs.earthdata.nasa.gov/twms/epsg4326/best/twms.cgi?request=GetTileService"
```
The output looks like a real raster. It shows the size (16384x81920 pixel), location (global), RGB and multiple resolutions.
 
## Geting a handle to the raster

The content of the SUBDATASET_\<N>_NAME is the XML that we are looking for. We can save it into a file and check:
```
echo '<GDAL_WMS><Service name="TiledWMS"><ServerUrl>https://gibs.earthdata.nasa.gov/twms/epsg4326/best/twms.cgi?</ServerUrl><TiledGroupName>MODIS Terra CorrectedReflectance TrueColor tileset</TiledGroupName></Service></GDAL_WMS>' >MODIS_TERRA.xml
gdalinfo MODIS_TERRA.xml
```
