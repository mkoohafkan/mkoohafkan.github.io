---
layout:     post
title:      Downloading, extracting and reading files in R
date:       2014-02-12 12:00:00
summary:    R provides some simple tools for downloading files, unzipping archives and reading in data.
categories: codemonkey file-management r
commentIssueId: 16
redirect_from:
  - /codemonkey/web/r/2014/02/12/downloading-extracting-and-reading-files-in-r/
  - /codemonkey/file-management/r/2014/02/12/downloading-extracting-and-reading-files-in-r/
---

I was recently asked to develop some reservoir forecasting tools for the San Francisco Public Utilities Commission, partly in response to the severe drought conditions facing California. I was asked to develop some simple tools to provide a workflow for pulling streamflow data, computing flow duration curves and fitting probability distributions to the data. The data products would be used for reservoir management optimization and risk analysis.

The first step is to actually get the data. In my case, my sources are streamflow data provided by the USGS waterData package and ensemble forecasts developed by the California Nevada River Forecast Center (<a href="http://www.cnrfc.noaa.gov/ensemble_theory.php">CNRFC</a>). I'm not going to go over downloading data using the waterData package as the documentation is fairly well developed; in particular, I recommend the vignette hosted on <a href="http://cran.r-project.org/web/packages/waterData/index.html">CRAN</a>. The CNRFC data is accessible on the web, and the procedure is instructive for basic file management in R.

So, let's download the data. CNRFC trace data is available as .zip files that can be accessed via the url `http://www.cnrfc.noaa.gov/csv/xxxxxxxx12_CentralCoast_hefs_csv_daily.zip` 
where xxxxxxxx is a datestring of format YYYYMMDD. To download this file in R, we first have to create a placeholder file. Since I don't want to worry about cleaning up after myself and explicitly deleting the files I create, I'll use the built-in functions `tempfile()` and `tempdir()` to place the files in R's default temporary directory, and then download today's data:

{% highlight r %}
# get the file url
today = as.Date(Sys.time())
forecasturl = paste('http://www.cnrfc.noaa.gov/csv/', 
                    gsub('-', '', today),
                    '12_CentralCoast_hefs_csv_daily.zip', sep='')
# create a temporary directory
td = tempdir()
# create the placeholder file
tf = tempfile(tmpdir=td, fileext=".zip")
# download into the placeholder file
download.file(forecasturl, tf)
{% endhighlight %}

The data has now been downloaded to a temporary file, and the full path is contained in `tf`. You'll notice I've also explicitly created a temporary directory, which I'll use for extracting the data from the archive. The `unzip()` function can be used to query the contents of a zip file and extract files to a specified location. In my case, the zip file will always contain a single file in CSV format.

{% highlight r %}
# get the name of the first file in the zip archive
fname = unzip(tf, list=TRUE)$Name[1]
# unzip the file to the temporary directory
unzip(tf, files=fname, exdir=td, overwrite=TRUE)
# fpath is the full path to the extracted file
fpath = file.path(td, fname)
{% endhighlight %}

A word of caution: if you are using an older version of R (i.e. older than v3.0.2), `fname` will actually be returned as a factor (this is a <a href="http://r.789695.n4.nabble.com/Fwd-R-unzip-method-gives-filenames-as-character-td4663767.html">bug</a>) and you will need to convert it to a string using `as.character()` before passing it to the second call to `unzip()`. 

Now I have extracted the file to the temporary directory, and I'm spoiled for choice when it comes to text reading functions in R. I'm going to use `read.csv()` since I know my file is in CSV format. However, my data files have two header lines and this creates a problem: all the data in the dataframe produced by `read.csv()` will be in string format because the second header line will be considered as data. I therefore need to do a little bit of extra formatting work. In addition, I need to make sure that I pass `stringsAsFactors=FALSE` to `read.csv()`; one of R's quirks is that the numeric value of factor is not the same as that of a string, e.g. `as.numeric(factor('6')) != as.numeric('6')`.

{% highlight r %}
# stringsAsFactors=TRUE will screw up conversion to numeric!
d = read.csv(fpath, header=TRUE, row.names=NULL, 
             stringsAsFactors=FALSE)
# drop 1st row (second header row)
forecast = d[seq(2, nrow(d)), ]
# first column is date, column name is 'GMT'
forecast[, 'GMT'] = as.Date(substr(forecast$GMT, 1, 10))
# all other columns are numeric
flowcols = seq(2, ncol(forecast)) 
forecast[, flowcols] = as.numeric(as.matrix(forecast[, flowcols]))
{% endhighlight %}

Now I have an quick way of getting CNRFC data into R, and the next time I close R the files in the temporary directory will automatically be cleaned up. Easy, right?