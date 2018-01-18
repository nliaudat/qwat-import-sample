# Importing data into Qwat – The Qgis expression way

## Abstract
<img align="right" width="132" height="184" src="https://github.com/nliaudat/qwat-import-sample/raw/master/documentation/imgs/virtual_field.png">

The approach chosen is to add virtual field to our initial data, using [Qgis expressions](https://docs.qgis.org/2.18/fr/docs/user_manual/working_with_vector/expression.html) to adapt the values to Qwat data model in a graphical way. 

The advantage is no need to make complex postgresql queries and errors can be corrected on the fly before final import. 

A simple copy/paste between initial and qwat's layers can be used. That solution works for small dataset and was successfully applied for approx 10000 pipes. You can also import the data into postgis and insert it to the corresponding views.


## Prerequisites
<img align="right" width="500" height="280" src="https://github.com/nliaudat/qwat-import-sample/raw/master/documentation/imgs/district.png">

Some layers need to be filled before importing any data.

Populate the distributor, district and pressurezone tables with your own data. 

In early step, district and pressurezone can be town limits.

Keep in mind the distributor's id. It will be used in imports later.
If you want custom id, it's recommended to keep values > 10000

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

### Optional : Insert [additional_pipe_material.sql](https://github.com/nliaudat/qwat-import-sample/raw/master/sample_virtual_fields/additional_pipe_material.sql) with pgAdmin : 
```
INSERT INTO qwat_vl.pipe_material (id, vl_active, short_fr, value_fr, diameter, diameter_nominal, diameter_internal, diameter_external, pressure_nominal) VALUES (10007,true,'PVC','Chlorure de polyvinyle',160,160,123.4,160,null);
....
```

You'll perhaps have to add your own or adapt. 
For small changes, you could directly add pipe material by editing the corresponding auxiliary table

<img width="446" height="203" src="https://github.com/nliaudat/qwat-import-sample/raw/master/documentation/imgs/pipe_material.png">



## Loading initial data

<img align="right" width="274" height="210" src="https://github.com/nliaudat/qwat-import-sample/raw/master/documentation/imgs/aquafri_tables.png">

As sample, we use AquaFri esri file geodatabase which is use in canton Fribourg.
Load the MN95 converted AquaFri gdb into qgis and save all tables as distinct geopackage in UTF8. 

The “Topology checker” plugin can help correcting bad or duplicate geometries.

### Optional : 3D mapping

If you have no or incomplete altimetric data, you could use a DEM to approximate the missing values.

For pipes, you could convert data to 3D with the grass [v.drape](https://grass.osgeo.org/grass72/manuals/v.drape.html) function. As example you can play with the scale factor (0.99825 if you want subtract approx. 1.4m from the DEM at 800m). 

For points (hydrant, valve, etc) you could use [Points sampling tool plugin](https://github.com/borysiasty/pointsamplingtool) to get missing Z values in another layer, and join it to the layer. Update the corresponding fk_precisionalti field to say that the altimetric value is approximated.

## The Virtual field 

<img width="471" height="426" src="https://github.com/nliaudat/qwat-import-sample/raw/master/documentation/imgs/add_virtual_field.png">

You can add the mandatory virtual field in layer properties.

https://rawgit.com/qwat/qwat-data-model/master/diagram/index.html give the Qwat database schema, with fields description.

You can list your distinct existing field value directly in Qgis expression editor

<img width="393" height="285" src="https://github.com/nliaudat/qwat-import-sample/raw/master/documentation/imgs/existing_field_values.png">

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

Look at the [sample import data](https://github.com/nliaudat/qwat-import-sample/tree/master/sample_virtual_fields) for a working list of mandatory field.

You need to respect some import order  (valve must clip on pipes, etc)


## Copy/paste expression fields

Qgis do not allow to copy virtual fields and expressions between layers.

You can do that by editing the *.qgs project file and copy/paste the <expressionfields> tags. 

Here is an example for hydrant (taken from [sample.qgs](https://github.com/nliaudat/qwat-import-sample/tree/master/sample_virtual_fields/sample.qgs))
```
      <expressionfields>
        <field typeName="int2" precision="0" expression="103" length="-1" type="2" comment="" name="fk_provider"/>
        <field typeName="int2" precision="0" expression="103" length="-1" type="2" comment="" name="fk_model_sup"/>
        <field typeName="int2" precision="0" expression="103" length="-1" type="2" comment="" name="fk_model_inf"/>
        <field typeName="int2" precision="0" expression="103" length="-1" type="2" comment="" name="fk_material"/>
        <field typeName="int2" precision="0" expression="103" length="-1" type="2" comment="" name="fk_output"/>
        <field typeName="int2" precision="0" expression="0" length="-1" type="2" comment="" name="underground"/>
        <field typeName="int2" precision="0" expression="null" length="-1" type="2" comment="" name="marked"/>
        <field typeName="double" precision="5" expression="" length="6" type="6" comment="" name="pressure_static"/>
        <field typeName="double" precision="5" expression="" length="6" type="6" comment="" name="pressure_dynamic"/>
        <field typeName="double" precision="5" expression="" length="6" type="6" comment="" name="flow"/>
        <field typeName="int2" precision="0" expression="null" length="-1" type="2" comment="" name="observation_date"/>
        <field typeName="text" precision="-1" expression="null" length="-1" type="10" comment="" name="observation_source"/>
        <field typeName="text" precision="-1" expression="" length="-1" type="10" comment="" name="identification"/>
        <field typeName="int2" precision="0" expression="" length="-1" type="2" comment="" name="fk_distributor"/>
        <field typeName="int2" precision="0" expression="" length="-1" type="2" comment="" name="fk_status"/>
        <field typeName="int2" precision="0" expression="" length="-1" type="2" comment="" name="fk_folder"/>
        <field typeName="int2" precision="0" expression="null" length="-1" type="2" comment="" name="fk_locationtype"/>
        <field typeName="int2" precision="0" expression="" length="-1" type="2" comment="" name="fk_precision"/>
        <field typeName="int2" precision="0" expression="1121" length="-1" type="2" comment="" name="fk_precisionalti"/>
        <field typeName="int2" precision="0" expression="103" length="-1" type="2" comment="" name="fk_object_reference"/>
        <field typeName="int2" precision="0" expression="" length="-1" type="2" comment="" name="fk_pressurezone"/>
        <field typeName="int2" precision="0" expression="" length="-1" type="2" comment="" name="year"/>
        <field typeName="int2" precision="0" expression="null" length="-1" type="2" comment="" name="year_end"/>
        <field typeName="int2" precision="0" expression="" length="-1" type="2" comment="" name="fk_district"/>
        <field typeName="int2" precision="0" expression="null" length="-1" type="2" comment="" name="fk_pressurezone"/>
        <field typeName="string" precision="0" expression="" length="500" type="10" comment="" name="remark"/>
      </expressionfields>
  ```    
  
  


## Tricks

### Import speed
To speed up importation, you can disable the logging trigger with pgAdmin

<img width="273" height="250" src="https://github.com/nliaudat/qwat-import-sample/raw/master/documentation/imgs/disable_trigger.png">

### Postgis copy import table

```
  -- better to remove old id
ALTER TABLE import.pipe DROP COLUMN id;

-- make sur geometry column have same name as destination
ALTER TABLE import.pipe RENAME geom TO geometry;
ALTER TABLE import.pipe ALTER COLUMN geometry TYPE public.geometry;
	
--for pipes work directly on the qwat_od.pipe table, some others on view.
INSERT INTO qwat_od.pipe(geometry,fk_function, fk_installmethod, fk_material, fk_distributor, fk_precision, fk_bedding, fk_protection, fk_status, fk_watertype, fk_folder, year, year_rehabilitation, year_end, pressure_nominal, remark, label_1_text, fk_district, fk_pressurezone)
SELECT geometry, fk_function, fk_installmethod, fk_material, fk_distributor, fk_precision, fk_bedding, fk_protection, fk_status, fk_watertype, fk_folder, year, year_rehabilitation, year_end, pressure_nominal, remark, label_1_text, fk_district, fk_pressurezone 
FROM import.pipe

-- be patient ;) - INSERT 10774 ==> Query returned successfully in 8 min.

  ```
