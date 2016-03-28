---
layout:     post
title:      Digging into GRASS with Python
date:       2013-08-10 12:00:00
summary:    Using code to leverage GIS tools is tricky, but rewarding.
categories: codemonkey gis python
redirect_from:
  - /codemonkey/gis/python/2013/08/10/digging-into-GRASS-with-python/
---

One of my projects at the <a href="http://www.blueoakranchreserve.org/">Blue Oak Ranch Reserve</a> involves making spatial predictions of evapotranspiration. Many of the ET models (such as Priestly-Taylor) require solar radiation as input, which means I need solar radiation maps to perform the calculations. Both <a href="http://www.esri.com/software/arcgis">ArcGIS</a> and <a href="http://grass.osgeo.org/">GRASS</a> offer solar radiation models, but I find GRASS advantageous because the documentation is well-developed and the radiation model can incorporate a number of features that the ArcGIS radiation model cannot, such as terrain shadowing.

When I say the documentation is well-developed, I mean the command manuals in the <a href="http://grass.osgeo.org/grass64/manuals/">function reference</a> specifically; documentation for setup and GRASS in general leaves a lot to be desired. My biggest issue was that I needed to run the radiation model for a number of days of the year, and running it manually would be tedious and time-consuming. I wanted to develop a simple loop that would run the radiation model for a specified sequence of days. GRASS supports this kind of functionality; in addition to being callable in text mode, it also interfaces with both <a href="http://grasswiki.osgeo.org/wiki/R_statistics">R</a> and <a href="http://grass.osgeo.org/programming6/pythonlib.html">Python</a>.

Unfortunately, I could not get much of anything to run using Bash or R. A combination of complicated setup requirements, poorly-developed help files and my limited experience with GRASS resulted in repeated failures. I couldn't even set up the environment variables properly to initialize GRASS and create a new mapset. But I did find a way with Python! It's a bit primitive, but hey---it works, and that's all I care about at this point.

I'm going to assume you know how to set up a mapset in GRASS and add/import some layers; the <a href="http://grass.osgeo.org/grass64/manuals/helptext.html">documentation</a> will get you that far at least. The tricky part can be the region setting, but I made things easy for myself by defining the extent based on my DEM. The next thing I did was run <a href="http://grass.osgeo.org/grass65/manuals/r.slope.aspect.html">r.slope.aspect</a>, since <a href="http://grass.osgeo.org/grass64/manuals/r.sun.html">r.sun</a> (the radiation model) uses slope and aspect as well as elevation to computer solar radiation.

If you're in the Layer Manager window, you should see the Python Shell tab at the bottom of the window. Going to this tab gives you a Python interface, and you can get access to GRASS functions by importing the <a href="http://grass.osgeo.org/programming6/pythonlib.html">grass.script module</a>. The module includes the `run_command()` function, which can be used to run basically any GRASS function. It supports flags through the <em>flags</em> argument, as well as any other named arguments relevant to the function you're calling.

So let's keep things simple and just run a script by first navigating to its parent folder (using `chdir()` from the <a href="http://docs.python.org/2/library/os.html">os module</a>), and then calling it using `execfile()`:

{% highlight python %}
import os
os.chdir("C:\myfolder\")
execfile("myscript.py")
{% endhighlight %}

And the script itself:

{% highlight python %}
import grass.script as grass

# i is the day of year (DOY) to use for running r.sun
for i in [51, 65, 79, 93, 107, 121, 135, 149, 161, 172, 182, 191, 205, 219, 
          233, 247, 257, 266, 278, 289, 303, 317, 331, 344, 356, 365]:
	# want to get insolation time, global radiation
	insolname = "insol_DOY" + str(i) + "_16ft"
	radname = "globrad_DOY" + str(i) + "_16ft"
	# run the solar model
	grass.run_command("r.sun", flags = "s",
			  elevin = "BORR_DEM16ft@PERMANENT",
			  aspin = "aspect_16ft@PERMANENT",
			  slopein = "slope_16ft@PERMANENT",
			  insol_time = insolname, glob_rad = radname, day = i)
	# export the outputs to GTiff files
	grass.run_command("r.out.gdal", input = insolname + "@PERMANENT", output = insolname)
	grass.run_command("r.out.gdal", input = radname + "@PERMANENT", output = radname)
{% endhighlight %}

And that's it! The script is running as I type this, and I'm free to write about it on my blog without having to stop every 15 minutes to set up another run of the solar radiation model.