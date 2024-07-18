# Land-Use-Change-Detection-Brandenburg
## Landsat 8 Water Analysis for Brandenburg

This repository contains a Google Earth Engine (GEE) script that analyzes water coverage changes in the Brandenburg region using Landsat 8 surface reflectance data for the years 2015 and 2023. The analysis calculates the Normalized Difference Water Index (NDWI) to identify water bodies and quantifies changes in water coverage over time.

## Introduction
This project aims to monitor and quantify changes in water coverage in Lausitz region, Germany. By using Landsat 8 surface reflectance data and the NDWI, we can detect water bodies and analyze their changes between 2015 and 2023.

## Data
The analysis utilizes Landsat 8 Collection 2 surface reflectance data:
- Time Periods: June 1, 2015 - July 31, 2015, and June 1, 2023 - July 31, 2023
- Bands: The script uses bands SR_B3 and SR_B5 for NDWI calculation.
- Cloud Cover Threshold: Images with less than 20% cloud cover are used.

## Methodology
1. Study Area Definition: The boundary of Brandenburg is defined using the FAO GAUL dataset.
2. Cloud Masking: A function masks clouds and cloud shadows in Landsat 8 data.
3. NDWI Calculation: The NDWI is calculated for both 2015 and 2023.
4. Water Detection: Pixels with NDWI values greater than 0.1 are classified as water.
5. Area Calculation: The area of water bodies is calculated for both years.
6. Change Detection: Changes in water coverage from 2015 to 2023 are analyzed and visualized.

## Results
The script outputs the following:
- Water coverage in hectares for 2015 and 2023.
- Area of land that changed to water from 2015 to 2023.
- Visualization layers for NDWI, water masks, and land use changes.

## Usage
### Prerequisites
- Google Earth Engine account
- Google Drive account

## Steps
1. Upload the Script to GEE: Copy the provided script into the GEE Code Editor.
2. Run the Script: Execute the script to process the data.
3. Review Outputs: The script will print the water areas and generate visualization layers.
4. Export Results: The script exports the results to your Google Drive.

```
// Define the study area: Brandenburg state boundary
var brandenburg = ee.FeatureCollection('FAO/GAUL_SIMPLIFIED_500m/2015/level2')
                   .filter(ee.Filter.eq('ADM1_NAME', 'Brandenburg'))
                   .geometry();

// Function to mask clouds in Landsat 8 Collection 2
function maskL8sr(image) {
  var cloudShadowBitMask = (1 << 4);
  var cloudsBitMask = (1 << 3);
  var qa = image.select('QA_PIXEL');
  var mask = qa.bitwiseAnd(cloudShadowBitMask).eq(0)
                   .and(qa.bitwiseAnd(cloudsBitMask).eq(0));
  return image.updateMask(mask).multiply(0.0000275).add(-0.2);
}

// Load Landsat 8 Collection 2 surface reflectance data for 2015
var landsat2015 = ee.ImageCollection('LANDSAT/LC08/C02/T1_L2')
                    .filterDate('2015-06-01', '2015-07-31')
                    .filterBounds(brandenburg)
                    .filter(ee.Filter.lt('CLOUD_COVER', 20))
                    .map(maskL8sr)
                    .median()
                    .clip(brandenburg);

// Load Landsat 8 Collection 2 surface reflectance data for 2023
var landsat2023 = ee.ImageCollection('LANDSAT/LC08/C02/T1_L2')
                    .filterDate('2023-06-01', '2023-07-31')
                    .filterBounds(brandenburg)
                    .filter(ee.Filter.lt('CLOUD_COVER', 20))
                    .map(maskL8sr)
                    .median()
                    .clip(brandenburg);

// Print available bands for Landsat 8
print('Landsat 8 bands:', landsat2015.bandNames());

// Define a function to calculate the Normalized Difference Water Index (NDWI) for Landsat 8
function addNDWI_L8(image) {
  return image.addBands(image.normalizedDifference(['SR_B3', 'SR_B5']).rename('NDWI'));
}

// Add NDWI to the images
var ndwi2015 = addNDWI_L8(landsat2015).select('NDWI');
var ndwi2023 = addNDWI_L8(landsat2023).select('NDWI');

// Define a threshold to classify water and non-water
var waterThreshold = 0.1;

// Create water/non-water masks for both years
var water2015 = ndwi2015.gt(waterThreshold);
var water2023 = ndwi2023.gt(waterThreshold);

// Define pixelArea
var pixelArea = ee.Image.pixelArea().divide(10000); // Pixel area in hectares

// Calculate the area of water in 2015
var waterArea2015 = water2015.multiply(pixelArea).reduceRegion({
  reducer: ee.Reducer.sum(),
  geometry: brandenburg,
  scale: 30,
  maxPixels: 1e9
}).get('NDWI');

// Calculate the area of water in 2023
var waterArea2023 = water2023.multiply(pixelArea).reduceRegion({
  reducer: ee.Reducer.sum(),
  geometry: brandenburg,
  scale: 30,
  maxPixels: 1e9
}).get('NDWI');

// Print the water area for both years
print('Water area in 2015 (hectares):', waterArea2015);
print('Water area in 2023 (hectares):', waterArea2023);

// Calculate land use change from non-water to water
var landUseChangeToWater = water2023.and(water2015.not());

// Calculate the area of land use change to water in hectares
var landUseChangeToWaterArea = landUseChangeToWater.multiply(pixelArea).reduceRegion({
  reducer: ee.Reducer.sum(),
  geometry: brandenburg,
  scale: 30,
  maxPixels: 1e9
}).get('NDWI');

// Print the area of land use change to water
print('Area of land use change to water (hectares):', landUseChangeToWaterArea);

// Visualization
Map.centerObject(brandenburg, 8);
var water2015Vis = water2015.updateMask(water2015).visualize({palette: ['blue'], min: 0, max: 1});
var water2023Vis = water2023.updateMask(water2023).visualize({palette: ['cyan'], min: 0, max: 1});
var landUseChangeToWaterVis = landUseChangeToWater.updateMask(landUseChangeToWater)
                                                  .visualize({palette: ['red'], min: 0, max: 1});
var ndwi2015Vis = ndwi2015.visualize({min: -1, max: 1, palette: ['00FFFF', '0000FF']});
var ndwi2023Vis = ndwi2023.visualize({min: -1, max: 1, palette: ['00FFFF', '0000FF']});

Map.addLayer(brandenburg, {}, 'Brandenburg Boundary');
Map.addLayer(ndwi2015Vis, {}, 'NDWI 2015');
Map.addLayer(ndwi2023Vis, {}, 'NDWI 2023');
Map.addLayer(water2015Vis, {}, 'Water 2015');
Map.addLayer(water2023Vis, {}, 'Water 2023');
Map.addLayer(landUseChangeToWaterVis, {}, 'Land Use Change to Water 2015-2023');

// Export the results to Google Drive
Export.image.toDrive({
  image: ee.Image().paint(brandenburg, 0, 2),
  description: 'Brandenburg_Boundary',
  folder: 'EarthEngineExports',
  fileNamePrefix: 'Brandenburg_Boundary',
  region: brandenburg,
  scale: 30,
  crs: 'EPSG:4326'
});

Export.image.toDrive({
  image: ndwi2015,
  description: 'NDWI_2015',
  folder: 'EarthEngineExports',
  fileNamePrefix: 'NDWI_2015',
  region: brandenburg,
  scale: 30,
  crs: 'EPSG:4326'
});

Export.image.toDrive({
  image: ndwi2023,
  description: 'NDWI_2023',
  folder: 'EarthEngineExports',
  fileNamePrefix: 'NDWI_2023',
  region: brandenburg,
  scale: 30,
  crs: 'EPSG:4326'
});

Export.image.toDrive({
  image: water2015,
  description: 'Water_2015',
  folder: 'EarthEngineExports',
  fileNamePrefix: 'Water_2015',
  region: brandenburg,
  scale: 30,
  crs: 'EPSG:4326'
});

Export.image.toDrive({
  image: water2023,
  description: 'Water_2023',
  folder: 'EarthEngineExports',
  fileNamePrefix: 'Water_2023',
  region: brandenburg,
  scale: 30,
  crs: 'EPSG:4326'
});

Export.image.toDrive({
  image: landUseChangeToWater,
  description: 'Land_Use_Change_to_Water',
  folder: 'EarthEngineExports',
  fileNamePrefix: 'Land_Use_Change_to_Water',
  region: brandenburg,
  scale: 30,
  crs: 'EPSG:4326'
});
```

## Water 2015
![image](https://github.com/user-attachments/assets/4cc0baeb-1fb4-4bef-96cb-43593392cd61)

## Water 2023
![image](https://github.com/user-attachments/assets/3ea40f3b-5936-419a-9b37-8ea655fcc467)

## Land Use Change to Water 2015-2023
![image](https://github.com/user-attachments/assets/17cd9d6f-978d-4ab6-a06f-ee1b9871e33d)
