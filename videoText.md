# Generate Grid
```javascript
d3.json(kyCounties, function(ky) {
    svg.call(zoom);
    // LOAD COUNTIES AND PRODUCE CENTROIDS
    var counties = topojson.feature(ky, ky.objects.counties).features;
    var kyExtent = d3.geo.bounds(topojson.mesh(ky, ky.objects.counties, function(a, b) { return a === b; }));
    // REFORMAT BOUNDS FOR GENERATING POINT GRID
    kyExtent = [kyExtent[0][0],kyExtent[0][1],kyExtent[1][0],kyExtent[1][1]];
    // GENERATE POINT GRID
    var grid = turf.within(turf.pointGrid(kyExtent,gridRes,'kilometers'),turf.featurecollection(counties));
})
```
#Generate Centroids
```javascript
for(var i = 0; i < counties.length; i++) {
    centroids[i] = turf.point(
        turf.center(counties[i]).geometry.coordinates, 
        {"z" : counties[i].properties.population}
    );
}
```
#Calculate Default Search Radius
```javascript
for (i in counties) kyArea += turf.area(counties[i]) / 1000000;
r = Math.sqrt((7 * kyArea)/(Math.PI *  centroids.length));
```
#Calculate distance all centroids from each grid cell
```javascript
for (i in grid.features) {
    for (j in centroids) {
        var dist = turf.distance(grid.features[i],centroids[j],'kilometers');
        centroids[j].properties.d = dist;
    }
```
#Adjust search radius according to eqn. A
```javascript
centroids.sort(function(a,b) {
    return a.properties.d - b.properties.d; 
});

if (r < centroids[3].properties.d) {
    selCentroids = centroids.slice(0,4);
} else if (r > centroids[3].properties.d && r <= centroids[9].properties.d) {
    for (n in centroids) {
        if (r < centroids[n].properties.d) {
            selCentroids = centroids.slice(0,n);   
            break;
        }
    }
} else {
    selCentroids = centroids.slice(0,10);   
}
var r = Math.ceil(selCentroids[selCentroids.length-1].properties.d);
```
#For each centroid...
###Calculate distance...
```javascript
for (cent_i in selCentroids) {

    ix = turf.point([
        selCentroids[cent_i].geometry.coordinates[0],
        grid.features[i].geometry.coordinates[1]
        ]);
    dist_ix = turf.distance(grid.features[i], ix, 'kilometers');

    iy = turf.point([
        grid.features[i].geometry.coordinates[0],
        selCentroids[cent_i].geometry.coordinates[1]
        ]);
    dist_iy = turf.distance(grid.features[i], iy, 'kilometers');
```
###And weight...
```javascript
if ( dist_i > 0 && dist_i <= (r / 3)) {
    s_i = 1 / dist_i;   
} else if ( (dist_i > (r / 3)) && (dist_i <= Math.ceil(r))) {
    s_i = (27 / (4 * r)) * ((dist_i / r) - 1) * ((dist_i / r) - 1);
} else s_i = 0;
```
###Weight the influence of all other centroids based on their angle from C_i
```javascript
for (cent_j in selCentroids) {
    if (cent_j != cent_i) {
        dist_j = selCentroids[cent_j].properties.d;

        jx = turf.point([
            selCentroids[cent_j].geometry.coordinates[0],
            grid.features[i].geometry.coordinates[1]
            ]);
        dist_jx = turf.distance(grid.features[i], jx, 'kilometers');

        jy = turf.point([
            grid.features[i].geometry.coordinates[0],
            selCentroids[cent_j].geometry.coordinates[1]
            ]);
        dist_jy = turf.distance(grid.features[i], jy, 'kilometers');

        cos = ((dist_ix * dist_jx) + (dist_iy * dist_jy)) / (dist_i * dist_j);


        if ( dist_j > 0 && dist_j <= (r / 3)) {
            s_j = 1 / dist_j;   
        } else if ( dist_j > (r / 3) && (dist_j <= r)) {
            s_j = (27 / (4 * r)) * ((dist_j / r) - 1) * ((dist_j / r) - 1);
        } else s_j = 0;

        t_iNum += s_j * (1 - cos);
        t_iDen += s_j;
    }
}
```
###Calculate direction- and distance-weighted value of grid cell
```javascript
t_i = t_iNum/t_iDen;
w = s_i * s_i * (1 + t_i);

idwNum += w * selCentroids[cent_i].properties.z;
idwDen += w;
idw = idwNum/idwDen;
grid.features[i].properties.z = idw;
```
#Calculate breaks based on values of all cells
```javascript
var breaks = ss.jenks(valueArray, numClasses);
                
var textShade = d3.scale.quantile()
    .domain(breaks)
    .range(textSymbology);
```
#Using overprinting symbology
```javascript
if (numClasses == 1) {
    textSymbology = ["."];
} else if (numClasses == 2) {
    textSymbology = [".", "OXAV"];
} else if (numClasses == 3) {
    textSymbology = [".","O","OXAV"];
} else if (numClasses == 4) {
    textSymbology = [".","+","O","OXAV"];
} else if (numClasses == 5) {
    textSymbology = [".","+","O","O-","OXAV"];
} else if (numClasses == 6) {
    textSymbology = [".","+","X","O-","OX","OXAV"];
} else if (numClasses == 7) {
    textSymbology = [".","'","+","X","O-","OX","OXAV"];
} else if (numClasses == 8) {
    textSymbology = [".","'","+","X","O","O-","OX","OXAV"];
} else if (numClasses == 9) {
    textSymbology = [".","'","=","+","X","O","O-","OX","OXAV"];
} else if (numClasses == 10) {
    textSymbology = [".","'","-","=","+","X","O","O-","OX","OXAV"];
}
```