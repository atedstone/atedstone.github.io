---
layout: post
title: Annual bare ice presence maps for the Greenland Ice Sheet using Google Earth Engine and MODIS MOD09GA
---

As part of a project that I'm currently working on, I wanted to compare some supraglacial stream network delineations to maps of where enough melting occurs each year for bare ice to be revealed. This is a perfect task for Google Earth Engine.


## Load MODIS imagery and area of interest

The code block below was generated automatically. For MOD09GA, the daily MODIS product I'm using, I looked it up in the inventory using the main search bar. 

For the geometry, I used the rectangle drawing tool within the Map part of the Google Earth Engine editor (code.earthengine.google.com). The rectangle covers the whole ice sheet.

```javascript
var mod09ga = ee.ImageCollection("MODIS/006/MOD09GA");
var geometry = 
    /* color: #d63000 */
    /* shown: false */
    /* displayProperties: [
      {
        "type": "rectangle"
      }
    ] */
    ee.Geometry.Polygon(
        [[[-75.59784235368275, 83.81279329413161],
          [-75.59784235368275, 58.9131234725084],
          [-7.218936103682753, 58.9131234725084],
          [-7.218936103682753, 83.81279329413161]]], null, false);
```

## Set basic temporal bounds

Here, I'm interested in generating bare ice maps from 2013 to 2020. Because melt only occurs in summer, there's little need include autumn-winter-spring months in the processing, so `ee.Filter.calendarRange` allows us to filter the image collection to return only July and August imagery. 

```javascript
var collection = mod09ga
    .filterBounds(geometry)
    .filterDate('2013-06-01', '2020-08-15')
    .filter(ee.Filter.calendarRange(7,8,'month'));
```    

## Generate annual maps

First, here's the function that identifies bare ice presence in all MODIS acquisitions. It scales the imagery by the MOD09GA documented scaling factor (0.0001) then applies the threshold of less than 0.6, as used previously (e.g. Tedstone et al., 2017, The Cryosphere).

```javascript
var bi = function(image) {
  var im = image.multiply(0.0001);
  im =  im.lt(0.6).and(image.gt(0));
  return im;
}
```

Now we map this function over every year of interest, to produce annual maps of bare ice presence. This script was thrown together quickly, so there's some other functionality hidden away in here too - selecting the band of interest (near infrared, 'b02') and clipping to the geometry that we defined previously.

```javascript
var years = ee.List.sequence(2013, 2020);
var bare_ice_annual_counts = ee.ImageCollection(years
  .map(function(y) {
    var start = ee.Date.fromYMD(y, 1, 1);
    var end = start.advance(12, 'month');
    var filtered = collection.filterDate(start, end).select('sur_refl_b02');
    var summed = filtered.map(bi).sum().clip(geometry);
    return summed;
}));
```

## Maps representative of the time period

```javascript
var bare_ice_mean = bare_ice_annual_counts.mean();
var bare_ice_max = bare_ice_annual_counts.max();

Map.addLayer(bare_ice_mean, {min:0, max:100}, 'composite_mean');
Map.addLayer(bare_ice_max, {min:0, max:100}, 'composite_max');
````

## Export to TIFFs

This includes reprojecting the outputs to North Polar Stereographic, EPSG 3413.

```javascript
Export.image.toDrive({image:bare_ice_mean, description:'annual_bare_ice_counts_mean_2013_2020', crs:'EPSG:3413'});

Export.image.toDrive({image:bare_ice_max.toDouble(), description:'annual_bare_ice_counts_max_2013_2020', crs:'EPSG:3413'});

var counts_as_img = bare_ice_annual_counts.toBands();
counts_as_img = counts_as_img.toDouble();
Export.image.toDrive({image:counts_as_img, description:'annual_bare_ice_counts', crs:'EPSG:3413'});
```



