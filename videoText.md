# Generate Grid
```javascript
d3.json(kyCounties, function(ky) {
    svg.call(zoom);
    // LOAD COUNTIES AND PRODUCE CENTROIDS
    var counties = topojson.feature(ky, ky.objects.counties).features;
    var kyExtent = d3.geo.bounds(topojson.mesh(ky, ky.objects.counties, function(a, b) { return a === b; }));
    // REFORMAT BOUNDS FOR GENERATING POINT GRID
    var kyExtent = [kyExtent[0][0],kyExtent[0][1],kyExtent[1][0],kyExtent[1][1]];
    // GENERATE POINT GRID
    var grid = turf.within(turf.pointGrid(kyExtent,gridRes,'kilometers'),turf.featurecollection(counties));
})
```
