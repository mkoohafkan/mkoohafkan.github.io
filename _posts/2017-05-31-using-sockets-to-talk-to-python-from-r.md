---
layout:     post
title:      Using sockets to talk to Python from R
date:       2017-05-31 12:30:00
summary:    I used sockets and JSON to build a bridge from R to Python.
categories: codemonkey r python json
commentIssueId: 27
---

A while back I developed the [arcpyr](https://github.com/mkoohafkan/arcpyr) 
package, which allows users to call ArcGIS tools from R. The package relies 
on the [PythonInR](https://cran.r-project.org/package=PythonInR) package to 
access the arcpy Python module. It works just fine... mostly. I've found 
that I actually need to use one of the non-exported functions in order to
reliably connect to the ArcGIS Python environment (which means my package 
won't pass CRAN checks) and I can't get it to work at all with the latest
version of ArcGIS Pro. PythonInR hasn't been updated in a long time, and
I don't have high hopes that it will start getting regular attention from 
the developer anytime soon.

On first glance, it looks like there are plenty of options to connect to
Python from R. In addition to the PythonInR package, there is the 
[RPython](https://cran.r-project.org/package=rPython) package and Yihui's 
[runr](https://github.com/yihui/runr) package. However, RPython hasn't 
really been tested on Windows and needs to be compiled with Python, so it
won't work with a pre-existing Python environment like the one provided by 
ArcGIS. The runr package looks promising, but it hasn't been updated in
years and I found it would hang when working with the ArcGIS Pro Python
environment.

Linking to Python from R can't be THAT hard, right? There are lots of options
for going the other way; Python has [rpy2](https://rpy2.bitbucket.io), 
[pyrserve](https://pypi.python.org/pypi/pyRserve) and
[pyper](https://pypi.python.org/pypi/PypeR/1.1.0). Surely I can figure out a way.

Using socket connections seemed to be the simplest solution. The runr
package uses sockets to pass messages between R and Python and a 
temporary file to write code and Python outputs. I decided to model
my package after runr because a) the code was readable and b) the server
script was mostly constructed, but I wasn't keen on the whole token file
thing. I spent a lot of time reading the Python socket
[documentation](https://docs.python.org/3/library/socket.html) and
[help](https://docs.python.org/2/howto/sockets.html). The big takeaway for me
was that a socket connection is for one-time use only, so the best way to 
indicate that you're finished sending data is to close the socket. 

I decided to stick with blocking sockets, because I want the Python code to
operate like a part of the R script and not force the user to script for
asynchronous server connections (keep it simple!). One requirement for me 
is that the code needs to work with both Python v2 and v3 (since ArcGIS Desktop
uses Python v2 but ArcGIS Pro uses Python v3), so I had to be wary 
of module imports and code formatting (mostly with respect to 
print statements and [io](https://docs.python.org/3/library/io.html) vs
[StringIO](https://docs.python.org/2/library/stringio.html)). I ended up 
[completely rewriting](https://github.com/mkoohafkan/pysockr/blob/master/inst/py-src/server.py) 
the Python server script from runr, reworked it multiple times and 
eventually came up with something almost identical to the original 
[server script](https://github.com/yihui/runr/blob/master/inst/lang/python_socket.py). 

The bigger differences are on the R side of things; I use an 
[R6 class](https://cran.r-project.org/web/packages/R6/vignettes/Introduction.html) 
to support multiple Python processes, with methods for 
starting/stopping the Python process and getting/setting variables.
The getting/setting variables bit is not trivial;
Python objects and R objects are not equivalent, some structures like lists and
vectors are similar but not identical, and determining the variable type from
a text string takes
extra effort. I didn't want to force the user to write their own code for parsing
results, but I didn't want to do it either! The solution turned out to be 
incredibly simple: use [JSON](https://www.json.org) as an intermediate format. 
Both Python and R support JSON via the 
[json](https://docs.python.org/3/library/json.html) module for Python and the 
[jsonlite](https://cran.r-project.org/package=jsonlite) package for R. My 
variable getting  and setting methods read/write JSON formats using `json.dumps` 
on the Python side and `toJSON`/`fromJSON` on the R side, so all the variable 
typing is done for me. If needed, users can define their own functions for 
encoding specific Python objects into JSON format.

I wrapped all this functionality into a package that I'm currently calling 
**pysockr**. 
It's [available on GitHub](https://github.com/mkoohafkan/pysockr) 
and ready for testing; install it with 
[devtools](https://cran.r-project.org/package=devtools), 
play around, and let me know what you think!
