---
layout:     post
title:      Using sockets to talk to Python from R
date:       2017-05-31 12:30:00
summary:    I used sockets and JSON to build a bridge from R to Python.
categories: codemonkey r python json
commentIssueId: 27
---

A while back I developed the [arcpyr](github.com/mkoohafkan/arcpyr) 
package, which allows users to call ArcGIS tools from R. The package relies 
on the [PythonInR](cran.r-project.org/package=PythonInR) package to 
access the arcpy Python module. It works just fine... mostly. I've found 
that I actually need to use one of the non-exported functions in order to
reliably connect to the ArcGIS Python environment (which means my package 
won't pass CRAN checks) and I can't get it to work at all with the latest
version of ArcGIS Pro. PythonInR hasn't been updated in a long time, and
I don't have high hopes that it start getting regular attention from the
developer anytime soon.

On first glance, it looks like there are plenty of options to connect to
Python from R. In addition to the PythonInR package, there is the 
[RPython](cran.r-project.org/package=rPython) package and Yihui's 
[runr](github.com/yihui/runr) package. However, RPython hsn't 
really been tested on Windows and needs to be compiled with Python, so it
won't work with a pre-existing Python environment like the one provided by 
ArcGIS. The runr package looks promising, but it hasn't been updated in
years and I found it would hang when working with the ArcGIS Pro Python
environment.

Linking to Python from R can't be THAT hard, right? There are lots of options
for going the other way; Python has [rpy2](rpy2.bitbucket.io), 
[pyrserve](pypi.python.org/pypi/pyRserve) and  
[pyper](pypi.python.org/pypi/PypeR/1.1.0). Surely I can figure out a way.

Usin socket connections seemed to be the simplest solution. The runr
package uses sockets to pass messages between R and Python and a 
temporary file to write code and Python outputs. I decided to model
my package after runr because a) the code was readable and b) the server
script was mostly constructed, but I wasn't keen on the whole token file
thing. I spent a lot of time reading the Python socket
[documentation](docs.python.org/3/library/socket.html) and
[help](https://docs.python.org/2/howto/sockets.html). The big takeaway for me
was that a socket connection is for one-time use only, so the best way to 
indicate that you're finished sending data is to close the socket. I also 
decided to stick with blocking sockets, because I want the Python code to
operate like a part of the R script and not force the user to script for
asynchronous serever connections (keep it simple!). One 
requirement for me was that the code had to work with both Python v2 and v3,
so I had to wary of my imports and code formatting (mostly with respect to 
print statements and [io](docs.python.org/3/library/io.html) vs
[StringIO](docs.python.org/2/library/stringio.html)). I ended up completely 
rewriting the Python server script from runr, reworked it multiple times and 
eventually came up with something almost identical to the original 
[server script](github.com/yihui/runr/blob/master/inst/lang/python_socket.py). 
The bigger differences are on the R side of things; I don't
support multiple independent Python environments in the same R session 
(similar to PythonInR) and I don't wrap everything up in an object, opting
instead for a collection of functions for starting/stopping the Python server
and getting/setting variables.

The getting/setting variables bit is not trivial. Python objects and R objects
are not equivalent, many structures like lists and vectors are similar 
(but not identical), and determining the variable type from a text string takes
extra effort. I didn't want to force the user to write their own code for parsing
results, but I didn't want to it either! The solution turned out to be incredibly 
simple: just use [JSON](www.json.org). Both Python and R support JSON via the 
[json](docs.python.org/3/library/json.html) module for Python and the 
[jsonlite](cran.r-project.org/package=jsonlite) package for R. My variable getting 
and setting functions read/write JSON formats, so all the variable typing is done 
for me. If needed, users can define their own functions for encoding specific Python 
objects into JSON format.

I wrapped all this functionality into a package that I'm currently calling 
**pysockr**. It's [available on GitHub](github.com/mkoohafkan/pysockr) and ready for
testing; install it with [devtools](cran.r-project.org/package=devtools), play around, 
and let me know what you think!
