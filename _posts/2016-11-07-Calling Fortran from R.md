---
layout:     post
title:      A student's guide to using Fortran in R
date:       2016-10-12 08:00:00
summary:    It's not hard to call Fortran from R. It's even easier if you have a student email account.
categories: codemonkey r fortran VisualStudio
---

A friend of mine has been working on a fairly large statistical analysis 
of flow and precipitation gauge data across California. R is the obvious
choice for this kind of work, but it can be rather slow when you need to 
crunch lots of numbers. I've [mentioned]() the [Rcpp]() package, which 
does a lot of the background work of connecting C++ code to R. However, 
Rcpp is [a bit limited]() compared to full C++, and it still requires
some technical know-how. 

There is a lot of Fortran code floating around out there, and while it 
pretty clunky and no fun at all to code, it's pretty easy to *use* 
existing code. You can use the GNU compiler to build Fortran 
applications and get by with reading and writing to files.

My friend had found one such piece of Fortran code: an autocorrelation
function that was sure to be faster than what R provides (despite the
fact that `acf` appears to be written in C). But because she needed to 
run this function on literally *hundreds* of datasets, she wanted to do
something a little more sophisticated (well, really just *faster*) than
doing a read-write-read loop with a Fortran executable.

It's actually not that hard to connect to Fortran from R. R 
provides the `dyn.load` function, which can be used to connect a DLL
file (e.g. a compiled Fortran program) to R, and the `.Fortran` 
function, which can 


It's not that



that my friend 
recently asked me A friend of mine recently asked me to help turn some 
old Fortran code into something useable by R. 


https://www.visualstudio.com/vs/community/

https://software.intel.com/en-us/qualify-for-free-software/student

https://www.visualstudio.com/vs/rtvs/


