## Spatial Join in AsterixDB

### New patch that adds rewritten rule for spatial join

Please download the path [here](add_rewritten_rule_for_spatial_join.patch)

### Create sample datasets for join queries<br/>

* Please make sure that you have a running instance of AsterixDB
* First, download the following ADM files as the input datasets for our tests:
[park_small.adm](park_small.adm), 
[lake_small.adm](lake_small.adm)
[park.adm](park.adm)
[lake.adm](lake.adm) 

* Second, run the following query to add the input datasets (park_small, lake_small) to the dataverse. 
You must replace the path "127.0.0.1:///home/tinvukhac/Documents/source_code/cartilage/spatialdatagenerators/output/*.adm" to the correct directory where you saved the files above.
```sql92
DROP DATAVERSE test IF EXISTS;
CREATE DATAVERSE test;
USE test;

-- Make Park type
CREATE TYPE ParkType as closed {
    id: int32,
    geom: rectangle
};

-- Make Lake type
CREATE TYPE LakeType as closed {
    id: int32,
    geom: rectangle
};

-- Make Park dataset 
CREATE DATASET ParkSet (ParkType) primary key id;
LOAD DATASET ParkSet USING localfs (("path"="127.0.0.1:///home/tinvukhac/Documents/source_code/cartilage/spatialdatagenerators/output/park.adm"),("format"="adm"));

-- Make Lake dataset 
CREATE DATASET LakeSet (LakeType) primary key id;
LOAD DATASET LakeSet USING localfs (("path"="127.0.0.1:///home/tinvukhac/Documents/source_code/cartilage/spatialdatagenerators/output/lake.adm"),("format"="adm"));
```  

### Run the spatial join query
```sql92
USE test;

SELECT COUNT(*) FROM ParkSet AS ps, LakeSet AS ls
WHERE spatial_intersect(ps.geom, ls.geom);
```

### Notes
* Currently, I disabled the join conditions in SpatialJoinRule after unnesting, just to make sure that the lost data cause by unnesting process.
* You can add these conditions back as follows:
```javascript
        // Compute reference tile ID
        ScalarFunctionCallExpression referenceTileId = new ScalarFunctionCallExpression(BuiltinFunctions.getBuiltinFunctionInfo(BuiltinFunctions.REFERENCE_TILE),
                new MutableObject<>(new VariableReferenceExpression(leftInputVar)),
                new MutableObject<>(new VariableReferenceExpression(rightInputVar)),
                new MutableObject<>(new ConstantExpression(new AsterixConstantValue(
                        new ARectangle(new APoint(MIN_X, MIN_Y), new APoint(MAX_X, MAX_Y))))),
                new MutableObject<>(new ConstantExpression(new AsterixConstantValue(new AInt64(NUM_ROWS)))),
                new MutableObject<>(
                        new ConstantExpression(new AsterixConstantValue(new AInt64(NUM_COLUMNS)))));

        // Update the join conditions with the tile Id equality condition
        ScalarFunctionCallExpression tileIdEquiJoinCondition =
                new ScalarFunctionCallExpression(BuiltinFunctions.getBuiltinFunctionInfo(BuiltinFunctions.EQ),
                        new MutableObject<>(new VariableReferenceExpression(leftTileIdVar)),
                        new MutableObject<>(new VariableReferenceExpression(rightTileIdVar)));
        ScalarFunctionCallExpression referenceIdEquiJoinCondition =
                new ScalarFunctionCallExpression(BuiltinFunctions.getBuiltinFunctionInfo(BuiltinFunctions.EQ),
                        new MutableObject<>(new VariableReferenceExpression(leftTileIdVar)),
                        new MutableObject<>(referenceTileId));
        ScalarFunctionCallExpression spatialJoinCondition = new ScalarFunctionCallExpression(
                BuiltinFunctions.getBuiltinFunctionInfo(BuiltinFunctions.SPATIAL_INTERSECT),
                new MutableObject<>(new VariableReferenceExpression(leftInputVar)),
                new MutableObject<>(new VariableReferenceExpression(rightInputVar)));
        ScalarFunctionCallExpression updatedJoinCondition =
                new ScalarFunctionCallExpression(BuiltinFunctions.getBuiltinFunctionInfo(BuiltinFunctions.AND),
                        new MutableObject<>(tileIdEquiJoinCondition),
                        new MutableObject<>(referenceIdEquiJoinCondition),
                        new MutableObject<>(spatialJoinCondition)
                );
        joinConditionRef.setValue(updatedJoinCondition);
```
