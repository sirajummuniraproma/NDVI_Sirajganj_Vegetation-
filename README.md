# NDVI_Sirajganj_Vegetation-
//Importing the dataset 
var img =ee.ImageCollection("COPERNICUS/S2_SR_HARMONIZED")
var boundary = ee.FeatureCollection("FAO/GAUL_SIMPLIFIED_500m/2015/level2")

//Area of interest 
var si = boundary.filter(ee.Filter.eq('ADM2_NAME','Sirajganj'))
Map.addLayer(si,{color:'green'},'Sirajganj Area')

//Making List
var month  = [1,2,3,4,5,6,7,8,9,10,11,12]
var year = 2021

//Loop function 
var fun = function(input){
  var start_date = ee.Date.fromYMD(year,input,01)
  var end_date =start_date.advance(1, 'month')
  
  var filter =img
  .filter(ee.Filter.date(start_date,end_date))
  .filterBounds(si.geometry())
  .filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE',55))
  .mean()
  .clip(si)
  
  print(filter)
  
  var veg = filter.normalizedDifference(['B8','B4'])
  var veg_con = veg.lt(0.9).and(veg.gt(0.5)).set('Year_Month',input)
  
  var total_veg = veg_con
  .multiply(ee.Image.pixelArea())
  .divide(1e6)
  .reduceRegion({
    reducer:ee.Reducer.sum(),
    geometry:si.geometry(),
    scale:100,
    maxPixels:1e13 })

  var feature = ee.Feature(null,{
    'start_time':start_date,
    'end_time':end_date,
    'total_veg_area':ee.Number(total_veg.get('nd')),
    'month_num':ee.Number(input),
    'year':year
  })
  
  return feature
  
}


var run = month.map(fun)
print(run)

//Feature Collection 
var feature_collection  = ee.FeatureCollection(run)

//Charting
var chart=ui.Chart.feature.byFeature(run,'month_num','total_veg_area')
.setChartType('LineChart')
  .setOptions({
    title: 'Monthly Vegetation Area in Sirajganj (2021)',
    titleTextStyle: {
      fontSize: 18,
      bold: true
    },
    hAxis: {
      title: 'Month',
      titleTextStyle: {italic: false, bold: true, fontSize: 14},
      // Use ticks to show month names instead of numbers
      ticks: [
        {v:1, f:'Jan'}, {v:2, f:'Feb'}, {v:3, f:'Mar'}, {v:4, f:'Apr'},
        {v:5, f:'May'}, {v:6, f:'Jun'}, {v:7, f:'Jul'}, {v:8, f:'Aug'},
        {v:9, f:'Sep'}, {v:10, f:'Oct'}, {v:11, f:'Nov'}, {v:12, f:'Dec'}
      ],
      gridlines: {count: 12}
    },
    vAxis: {
      title: 'Vegetation Area (km²)',
      titleTextStyle: {italic: false, bold: true, fontSize: 14},
      viewWindow: {min: 0},
      gridlines: {count: 10}
    },
    legend: {
      position: 'top'
    },
    lineWidth: 2,
    pointSize: 5,
    colors: ['#2ecc71'],
    series: {
      0: {lineWidth: 3, pointSize: 6}
    },
    interpolateNulls: true,
    curveType: 'function',
    fontSize: 12,
    chartArea: {
      backgroundColor: '#f9f9f9',
      left: 100,
      top: 50,
      width: '70%',
      height: '60%'
    }
  })

print('Chart:', chart)

//Export the Data
Export.table.toDrive({
  collection:feature_collection,
  description:'Sirajganj_VEG_Index_2021',
  folder:'GEE_si_DATA', 
  fileNamePrefix:'SI_VEG_2021',
  fileFormat:'CSV' })