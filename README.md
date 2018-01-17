# Importing data into Qwat – The Qgis expression way

## Abstract
The approach chosen is to add virtual field to our initial data, using qgis expression1 to adapt the values to qgis data model in a graphical way. 

The advantage is no need to make complex postgresql queries and errors can be corrected on the fly before final import. 

A simple copy/paste between initial and qwat's layers can be used. That solution works for small dataset and was successfully applied for approx 10000 pipes. You can also import the data into postgis and insert it to the corresponding views.
```
INSERT INTO qwat_od.pipe FROM my_pipe_import;
```

## Prerequisites
Some layers need to be filled before importing any data.

Populate the distributor, district and pressurezone tables with your own data. 

In early step, district and pressurezone can be town limits.

Keep in mind the distributor's id. It will be used in imports later.
If you want custom id, it's recommended to keep values > 100002

District and pressurezone linked id will be calculated in virtual fields using the following function : 
```
from qgis.core import *
from qgis.gui import *

@qgsfunction(args='auto', group='Custom')
def is_contained_in(layername, column, feature, parent):
    layer = QgsMapLayerRegistry.instance().mapLayersByName(layername)[0]
    for feat in layer.getFeatures():
        if feature.geometry().within(feat.geometry()):
            return feat[column]
```
You need to add that script in qgis expression editor in the function tab.

### Optional : Insert additional_pipe_material.sql with pgAdmin : 
```
INSERT INTO qwat_vl.pipe_material (id, vl_active, short_fr, value_fr, diameter, diameter_nominal, diameter_internal, diameter_external, pressure_nominal) VALUES (10007,true,'PVC','Chlorure de polyvinyle',160,160,123.4,160,null);
....
```

You'll perhaps have to add your own or adapt. 
For small changes, you could directly add pipe material by editing the corresponding auxiliary table



## Loading initial data

As sample, we use AquaFri esri file geodatabase which is use in canton Fribourg.
Load the MN95 converted 3AquaFri gdb into qgis and save all tables as distinct geopackage in UTF8. 4

The “Topology checker” plugin can help correcting bad or duplicate geometries.

### Optional : 3D mapping

If you have no or incomplete altimetric data, you could use a DEM to approximate the missing values.

For pipes, you could convert data to 3D with the grass v.drape function. As example you can play with the scale factor (0.99825 if you want subtract approx. 1.4m from the DEM at 800m). 

For points (hydrant, valve, etc) you could use “Points sampling tool plugin”5 to get missing Z values in another layer, and join it to the layer. Update the corresponding fk_precisionalti field to say that the altimetric value is approximated.

## The Virtual field 

You can add the mandatory virtual field in layer properties.

https://rawgit.com/qwat/qwat-data-model/master/diagram/index.html give the Qwat database schema, with fields description.

You can list your distinct existing field value directly in Qgis expression editor


A CASE WHEN expression suits for 95% of the job : 
```
CASE WHEN "etat_connexion" = 'ferme' THEN 't'
else 'f'
END
```

But you can enjoy the power of Expression, with custom function
```
is_contained_in( 'communes','id') 
```
and more 
```
dbquery( 'conduites - matériaux','id',  'diameter_external > 160 ')
```

Look at the sample import data for a working list of mandatory field.

You need to respect some import order  (valve must clip on pipes, etc)

