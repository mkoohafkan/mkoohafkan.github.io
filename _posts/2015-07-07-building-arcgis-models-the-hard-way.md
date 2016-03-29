---
layout:     post
title:      Building ArcGIS models the hard way
date:       2015-07-07 12:00:00
summary:    ArcGIS can be finicky. Model Builder even more so.
categories: codemonkey gis arcgis
redirect_from:
  - /codemonkey/gis/arcgis/2015/07/07/building-arcgis-models-the-hard-way/
---

I was recently contracted by the USACE Hydrologic Engineering Center to develop
an ArcGIS toolbox for working with some land cover geodatabases. Larry Strong
at the [Northern Prairie Wildlife Research Center](http://www.npwrc.usgs.gov/)
has developed some really detailed
[geomorphic object and land cover geodatabases](http://pubs.usgs.gov/fs/2007/3087/)
for investigating [emergent sandbar habitat](http://pubs.usgs.gov/of/2008/1223/),
but using the geodatabases can be tricky; issues regarding how to deal with
overlapping images, accounting for discharge at the time the image was taken
and extracting geomorphic objects based on fuzzy membership values
is hard enough when you're a GIS expert. My job was to make sure that you
*didn't* have to be one to work with the data.

Making an ArcGIS toolbox is the most obvious way of providing a toolset to
users; they are familiar to access and use and are easily documented and
organized. ArcGIS offers two ways to build models. The first method is to write
[Python](http://desktop.arcgis.com/en/analytics/python/) code that calls ArcGIS
tools via [ArcPy](http://resources.arcgis.com/en/help/main/10.2/#/A_quick_tour_of_ArcPy/000v00000001000000/)
(similar to the way one can write [scripts for GRASS GIS]({% post_url 2013-08-10-digging-into-GRASS-with-python %})).
The second method involves the [Model Builder](http://resources.arcgis.com/en/help/main/10.2/index.html#//002w00000028000000),
which provides a graphical user interface for chaining ArcGIS tools together.
Being an experienced programmer, I was naturally drawn to the first method.
However, I had an important constraint: because my contract was quite short, I
had to develop the tools in such a way that other employees would be able to
modify and build on them with minimum hassle. Since I could not count on
the next employee to be fluent in Python, I opted to create the toolbox using
Model Builder.

Model Builder is pretty impressive---The interface is clean and intuitive, and
it is easy to document your tools. However, there are a few things about it
that drive me insane.

#### Formatting the documentation is obnoxious

This isn't a big deal or anything, but it's extremely annoying that editing a
tool's  documentation often results in spaces being removed, whether they are
empty lines, paragraph indentations or (most aggravatingly) following a
bolded or italicized word. ArcGIS tries to be smart by copying formatting as
well as text; unfortunately it fails miserably, and copying and pasting usually 
results in exactly what you didn't want.

#### Pathnames will break everything when you try to share your toolbox

In exactly the same way that
[ArcGIS stores absolute instead of relative pathnames by default](http://www.northrivergeographic.com/arcgis-relative-paths-in-mxds),
Model Builder stores absolute rather than relative paths to tools by default.
This means that if you create a model that is in turn used by another model
(i.e. a [submodel](http://resources.arcgis.com/en/help/main/10.2/index.html#//002w0000007p000000)),
your model will not work if you want someone else to use it on their machine
(which you probably do, if you are bothering to build a model in the first
place). The solution is to go to the Model --> Model Properties menu option in
the Model Builder interface and check the box to store relative pathnames---and
yes, you have to do this for each model individually.

#### Error 000670 will make you cry (even when do the right thing)

Despite the fact that Model Builder allows you to tag some intermediate outputs 
as [managed](http://help.arcgis.com/en/arcgisdesktop/10.0/help/index.html#/Making_intermediate_data_managed_data/002w0000005p000000/)
(and therefore let ArcGIS automatically take care of naming and 
garbage collection) file names can still cause you grief. For example, if you 
want to run the same tool in a chain twice with different settings and then 
merge the results (e.g. you want to compute summary statistics over all 
categories, and also within each individual category) your model might break if 
the input file name is too long. This is because the default output names are 
based on the input, and if the 
[output name is too long](http://gis.stackexchange.com/questions/53244/filepath-length-errors-in-arcmap) 
Model Builder will truncate it---including the part of the name that 
distinguishes it from the output created the first time the tool was run. Oh, 
and don't think this is a problem for rasters only---vector file names may also 
get truncated sometimes.

#### Don't bother trying to run your models in batch

In their infinite wisdom (no, I'm not being sarcastic *at all*) The ArcGIS 
programmers thought that it would be a good idea to implement batch 
functionality in such a way that 
[each step in a model is run for every input in the batch](http://gis.stackexchange.com/questions/36891/does-calculate-value-model-only-tool-work-correctly-in-tools-run-in-batch) before moving on to the next step, rather than running the full model
for each input in the batch. This means that things like selection queries, 
field calculations and pretty much anything else important will silently break 
in batch mode and give you results that are complete garbage.

#### Despite all this

If you can get around these problems, you're in good shape to build models with
Model Builder. And honestly, despite these problems it's a pretty great tool; 
even if you have no programming experience whatsoever, you can still make 
complicated models that can simplify your workflow considerably.

