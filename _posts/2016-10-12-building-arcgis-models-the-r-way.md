---
layout:     post
title:      Building ArcGIS models the R way
date:       2016-10-12 08:00:00
summary:    My pathological obsession with R led me to develop an interface for ArcPy.
categories: codemonkey r pythoninr arcpy arcgis
commentIssueId: 22
---

I do a lot of GIS work, and ArcGIS is my tool of choice---mostly because 
all of my collaborators use it. While I do a lot of stuff through the ArcMap 
interface, there are times when I need to do things programmatically. 
Model Builder helps a lot if you don't do a lot of programming or need 
to do relatively simple tasks, but it can be pretty 
[finicky]({% post_url 2015-07-07-building-arcgis-models-the-hard-way %}) 
and requires some weird workarounds for things that should be simple. 

I frequently find myself faced with tasks that require complex chaining 
of ArcGIS functions. Consider, for example, applying area-weighted 
averaging as a dissolve statistic (which absolutely should be a standard 
method for the `Dissolve` tool, but isn't). In order to accomplish this, 
you need to:

  1. Create a new field which is the product of the shape area 
     and the field you want to calculate the area-weighted average for;
  2. Dissolve your feature class with the SUM statistic for your new 
     field; and
  3. Create a new field which is the SUM statistic divided by the 
     dissolved shape area.

This is a pretty simple procedure in terms of logic, but it gets fairly 
messy in Model Builder because of the multiple calls to `Add Field` and 
`Calculate Field`, and the parameter exposure needed in order to maintain
sensible field names. Doing it in `ArcPy` is much easier:

```python
def wadissolve(inlayer, outlayer, wafield, dissolvefield):
  # create the temporary field
  tempfield = wafield + '_PROD'
  prodexpr = '!' + wafield + '! * !SHAPE_AREA!'
  AddField_management(inlayer, tempfield, "DOUBLE")
  CalculateField_management(inlayer, tempfield, prodexpr, "PYTHON_9.3")
  # dissolve with the SUM statistics for the temporary field
  Dissolve_management(inlayer, outlayer dissolvefield, tempfield + ' SUM')
  # calculate the area-weighted average
  divexpr = '!' + tempfield + '_SUM! / !SHAPE_AREA!'
  AddField_management(outlayer, wafield, "DOUBLE")
  CalculateField_management(outlayer, awfield, divexpr, "PYTHON_9.3")
  # cleanup
  DeleteField_management(inlayer, tempfield)
  DeleteField_management(outlayer, tempfield + '_SUM')
  return outlayer
```

I have a problem though, and that problem is that I'd rather not use 
Python. I've mentioned before that I'm not a fan of the syntax, and 
there are some R packages that are indispensable for the types of 
analyses I'm doing. But there's no R interface for ArcGIS. What 
other choice do I have?

There is the [RPyGeo](https://cran.r-project.org/web/packages/RPyGeo) 
package, which claims to "provide access to 
(virtually any) ArcGIS Geoprocessing tool from within R by running 
Python geoprocessing scripts without writing Python code or touching 
ArcGIS". It works, but it doesn't really read like normal R code; except
for a small number of functions that the authors wrote interfaces for,
you call ArcPy functions through the `rpygeo.geoprocessor` with text 
strings and argument lists, and some specification of temporary python 
script files and message output files. I also don't understand the 
dependency on `RSAGA`, or even `shapefiles` for that matter. 

It turns out I'm stubborn enough to develop my own R interface for ArcPy 
rather than change my workflow a little bit. It seemed to me that the
simplest approach was to use a virtual Python environment from which to 
access ArcPy. The 
[PythonInR](https://cran.r-project.org/web/packages/PythonInR) 
package does this, and it does it well---automatically creating the 
function interfaces required with the `pyImport` function so that it 
behaves just like a normal R function (except the arguments are not 
exposed,everything takes `...`). There can be a little bit of weirdness
with return values (e.g. the text 'true' instead of the logical value 
TRUE) but this really isn't an issue for ArcPy, which generally returns 
the output file path as a string. The output file path is almost always
an input argument as well, so you usually don't need to access the 
return value.

I used PythonInR to build the package 
[arcpyr](https://github.com/mkoohafkan/arcpyr), which provides a pretty
straightforward interface to ArcPy. It's not a complete solution; it 
provides access to ArcPy functions, but it can't do a lot of the 
object-oriented stuff. I created a few functions to provide raster 
calculator and access some of the environment settings. But for examples 
like the one above, using ArcPy reads like any other R code:

```r
wadissolve = function(inlayer, outlayer, wafield, dissolvefield){
  # create the temporary field
  tempfield = paste0(wafield, '_PROD')
  prodexpr = paste0('!', wafield, '! * !SHAPE_AREA!')
  AddField_management(inlayer, tempfield, "DOUBLE")
  CalculateField_management(inlayer, tempfield, prodexpr, "PYTHON_9.3")
  # dissolve with the SUM statistics for the temporary field
  Dissolve_management(inlayer, outlayer dissolvefield, 
    paste0(tempfield, ' SUM'))
  # calculate the area-weighted average
  divexpr = paste0('!' , tempfield, '_SUM! / !SHAPE_AREA!')
  AddField_management(outlayer, wafield, "DOUBLE")
  CalculateField_management(outlayer, awfield, divexpr, "PYTHON_9.3")
  # cleanup
  DeleteField_management(inlayer, tempfield)
  DeleteField_management(outlayer, paste0(tempfield, '_SUM'))
  outlayer
}
```

Having an R interface for ArcPy means I don't have to juggle Python and
R together to do my analysis, Making it way easier to maintain my 
codebase and rerun my analyses when needed. Of course, I could also 
use PythonInR to run entire python scripts if I needed to leverage
the object-oriented aspects of ArcPy. One thing that got on my nerves 
was that loading the ArcPy functions using `PythonInR::pyImport` 
cluttered up the global R environment; I've now started a 
[new implementation](https://github.com/mkoohafkan/arcpyr/tree/arcpy-env) 
of the package that provides a cleaner interface.
