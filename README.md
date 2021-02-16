# GDAL-twms-with-NASA-GIBS
How to make use of the NASA/GIBS services using GDAL and the tWMS client

## WARNING

**This information is preliminary, not all this functionality is present in GDAL yet.**

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
It takes a few seconds and a lot of output will be generated. This is the output from the WMSMetaDriver, which recognized the response and parsed it. The dataset itself is just a placeholder, the real output is in the metadata, which exposes all the datasets available on the server as GDAL subdatasets. There are close to one thousand different datasets available on that server. Each one will get a **SUBDATASET_\<N>_NAME** and a **SUBDATASET_\<N>_DESC**, where \<N> is the subdataset number. The **SUBDATASET_NAME** is picked right out of the GetTileService response and should be a readable, informative description of the respective dataset. Let's pick subdataset 479 and take a closer look:
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
 
## Geting a handle to a single dataset

The content of the SUBDATASET_\<N>_NAME is the XML that we are looking for. We can save it into a file and check. Careful with the quotes, XML requires double quotes for attribute values.
```
  echo '<GDAL_WMS><Service name="TiledWMS"><ServerUrl>https://gibs.earthdata.nasa.gov/twms/epsg4326/best/twms.cgi?</ServerUrl><TiledGroupName>MODIS Terra CorrectedReflectance TrueColor tileset</TiledGroupName></Service></GDAL_WMS>' >MODIS_TERRA.xml
  gdalinfo MODIS_TERRA.xml
```

The XML string itself can be used as a handle , the result will be the same:
```
  gdalinfo '<GDAL_WMS><Service name="TiledWMS"><ServerUrl>https://gibs.earthdata.nasa.gov/twms/epsg4326/best/twms.cgi?</ServerUrl><TiledGroupName>MODIS Terra CorrectedReflectance TrueColor tileset</TiledGroupName></Service></GDAL_WMS>'
```

## Get the pixels

Now that we have a handle, getting the data is easy with gdal_translate:
```
  gdal_translate -of JPEG -outsize 2560 1280 MODIS_TERRA.xml MODIS_TERRA.jpg
```
It takes a few seconds and it's done. The parameters ask for the output to be a JPEG image and set the output size in pixels. The full resolution is too large to be saved in a single JPEG and would take a long time. The output image should look right, but will always have a large black area. That is because the default time is ***now***, which in the MODIS case it means ***today***. And since data for today is still being generated, only a part of the world will have data.

## Choose the date

GIBS specializes in providing temporal data. MODIS Terra has been operational for more than a dozen years and all the available data is in the GIBS server. Let's pick a random date, 2020-02-05, which is Feb 05 2020 in ISO 9601 notation, as required by the GIBS server. We can request any date using the same handle file:
```
  gdal_translate -of JPEG -outsize 2560 1280 -oo Change=time:2020-02-05 MODIS_TERRA.xml MODIS_T_Feb_05_2020.jpg
```
That is pretty simple, right? The -oo means OpenOption, a way to pass a string to the tWMS driver. This will only work with *time*, because it has to match what the server can handle. Otherwise an error will occur. If a date in the future is requested, an empty image will be generated, since no data is available.
So we get a pretty picture, but how do we open a specific day in a GIS application, for example in Esri's ArcPro?

## Save the date

The solution is to copy the raster into a WMS type raster:
```
  gdal_translate -of WMS -oo Change=time:2020-02-05 MODIS_TERRA.xml MODIS_T_Feb_05_2020.tWMS
```
This only works if the full tiledWMS input raster is directly copied into another tiledWMS raster. Otherwise a WMS format output file can't be generated. The size in pixels doesn't matter, since no pixels get copied, all we get is another XML handle to the GIBS dataset. Let's take a look at the content:
```
<GDAL_WMS>
  <Service name="TiledWMS">
    <ServerUrl>https://gibs.earthdata.nasa.gov/twms/epsg4326/best/twms.cgi?</ServerUrl>
    <TiledGroupName>MODIS Terra CorrectedReflectance TrueColor tileset</TiledGroupName>
    <Change Key="${time}">2020-02-05</Change>
  </Service>
</GDAL_WMS>
```
It makes sense, it is very similar to the MODIS_TERRA.xml, but tiledWMS stored the fact that the key ${time} has to be changed into 2020-02-05 when requesting data.
We can test that it work from the command line, or we can open that file in any GDAL based GIS. In ArcPro, the tWMS is not recognized without some extra configuration steps, but it can be dragged and dropped into a map and it will work. Or just change the extension into something supported, like *.tif* for example.

## Single handle for any dataset
The TiledGroupName can also be fed as an open option. First, we have to create a XML hook that doesn't contain the TiledGroupName nor the Change.
```
  echo '<GDAL_WMS><Service name="TiledWMS"><ServerUrl>https://gibs.earthdata.nasa.gov/twms/epsg4326/best/twms.cgi?</ServerUrl></Service></GDAL_WMS>' >GIBS_GCS.xml
```
Now we can generate XML handles for any dataset by name and specifying the time. Let's get the MODIS Aqua JPEG for the same day:
```
  gdal_translate -of JPEG -outsize 2560 1280 -oo TiledGroupName="MODIS Aqua CorrectedReflectance TrueColor tileset" -oo Change=time:2020-02-05 GIBS_GCS.xml MODIS_A_Feb_05_2020.jpg
```
Of course, both parameters can be saved in a tWMS file, just like before:
```
  gdal_translate -of WMS -oo TiledGroupName="MODIS Aqua CorrectedReflectance TrueColor tileset" -oo Change=time:2020-02-05 GIBS_GCS.xml MODIS_A_Feb_05_2020.tWMS
```

## Finding datasets easier

The parameter to the TiledGroupName has to be an exact match to what the server declares, white spaces included, which makes it tricky. Fortunately, there is a way to find the exact string, using the WMSMetaDriver.
Let's go back to the start:
```
  gdalinfo "https://gibs.earthdata.nasa.gov/twms/epsg4326/best/twms.cgi?request=GetTileService"
```
That lists all the datasets. We can use the TiledGroupName open option with a prefix string to restrict the listed datasets. For example, to get all the tiled group names that contain the substring "MODIS TERRA", we can use:
```
  gdalinfo -oo TiledGroupName="MODIR TERRA" "https://gibs.earthdata.nasa.gov/twms/epsg4326/best/twms.cgi?request=GetTileService"
```
That is a lot more manageable. The substring match is done case insensitive.

## Use gdal_translate to generate hook files

gdal_translate has a \-sds option where each subdataset is handled in sequence. This can be used to generate multiple hook files in a single command. So we can use the open options to restrict what gets generated. For example, generating all the hooks for patterns that contain the word infrared:
```
  gdal_translate -of WMS -sds -oo TiledGroupName="infrared" -oo Change=time:2019-10-21 "https://gibs.earthdata.nasa.gov/twms/epsg4326/best/twms.cgi?request=GetTileService" Infrared.tWMS
```
This will take a little while, and will generate a few files called Infrared_<N>.tWMS. It is not very efficient, since the output file is opened after creation, to verify that the file is valid. And each tiledWMS open has to fetch the GetTileService response from the server and parse it. Using gdalinfo, we can check what group name is in which file and that the time was captured, by looking at the metadata:
```
  gdalinfo Infrared_1.tWMS
```
  
## Eliminating the GetTileService request
It takes a few seconds for the GIBS server to reply to the GetTileService request. If multiple tiledWMS files are opened, this can be an unacceptable delay. This delay can be eliminated by storing the GetTileService response in the tWMS hook file. From the command line only one method is available, which stores the file inside the tWMS hook file itself. For example, we can modify the previous command and use:
```
gdal_translate -of WMS -sds -oo StoreConfiguration=yes -oo TiledGroupName="infrared" -oo Change=time:2019-10-21 "https://gibs.earthdata.nasa.gov/twms/epsg4326/best/twms.cgi?request=GetTileService" Infrared.tWMS
```
The generated files are a lot larger because they contain the XML encoded response to the GetTileService. But they might work faster, because the file does not have to be retrieved from the server when opening the file.
