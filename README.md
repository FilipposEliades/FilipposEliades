// MODULES DECLARATION -----------------------------------------------------------
// Total Precipitable Water 
var NCEP_TPW = require('users/sofiaermida/landsat_smw_lst:modules/NCEP_TPW.js')
//cloud mask
var cloudmask = require('users/sofiaermida/landsat_smw_lst:modules/cloudmask.js')
//Normalized Difference Vegetation Index
var NDVI = require('users/sofiaermida/landsat_smw_lst:modules/compute_NDVI.js')
//Fraction of Vegetation cover
var FVC = require('users/sofiaermida/landsat_smw_lst:modules/compute_FVC.js')
//surface emissivity
var EM = require('users/sofiaermida/landsat_smw_lst:modules/compute_emissivity.js')
// land surface temperature
var LST = require('users/sofiaermida/landsat_smw_lst:modules/SMWalgorithm.js')
// --------------------------------------------------------------------------------

var COLLECTION = ee.Dictionary({
  'L4': {
    'TOA': ee.ImageCollection('LANDSAT/LT04/C01/T1_TOA'),
    'SR': ee.ImageCollection('LANDSAT/LT04/C01/T1_SR'),
    'TIR': ['B6',]
  },
  'L5': {
    'TOA': ee.ImageCollection('LANDSAT/LT05/C01/T1_TOA'),
    'SR': ee.ImageCollection('LANDSAT/LT05/C01/T1_SR'),
    'TIR': ['B6',]
  },
  'L7': {
    'TOA': ee.ImageCollection('LANDSAT/LE07/C01/T1_TOA'),
    'SR': ee.ImageCollection('LANDSAT/LE07/C01/T1_SR'),
    'TIR': ['B6_VCID_1','B6_VCID_2'],
  },
  'L8': {
    'TOA': ee.ImageCollection('LANDSAT/LC08/C01/T1_TOA'),
    'SR': ee.ImageCollection('LANDSAT/LC08/C01/T1_SR'),
    'TIR': ['B10','B11']
  }
});

exports.collection = function(landsat, date_start, date_end, geometry, use_ndvi){


  // load TOA Radiance/Reflectance
  var collection_dict = ee.Dictionary(COLLECTION.get(landsat));

  var landsatTOA = ee.ImageCollection(collection_dict.get('TOA'))
                .filter(ee.Filter.date(date_start, date_end))
                .filter(ee.Filter.eq('WRS_PATH', 176))
                .filter(ee.Filter.eq('WRS_ROW', 36))
                .filterBounds(geometry)
                .map(cloudmask.toa);
  
              
  // load Surface Reflectance collection for NDVI
  var landsatSR = ee.ImageCollection(collection_dict.get('SR'))
                .filter(ee.Filter.date(date_start, date_end))
                .filterBounds(geometry)
                .filter(ee.Filter.eq('WRS_PATH', 176))
                .filter(ee.Filter.eq('WRS_ROW', 36))
                .map(cloudmask.sr)
                .map(NDVI.addBand(landsat))
                .map(FVC.addBand(landsat))
                .map(NCEP_TPW.addBand)
                .map(EM.addBand(landsat,use_ndvi));

  // combine collections
  // all channels from surface reflectance collection
  // except tir channels: from TOA collection
  // select TIR bands
  var tir = ee.List(collection_dict.get('TIR'));
  var landsatALL = (landsatSR.combine(landsatTOA.select(tir),true));
  
  // compute the LST
  var landsatLST = landsatALL.map(LST.addBand(landsat));

  return landsatLST;
};
