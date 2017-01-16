---
layout:     post
title:      Leveraging laboratory data with R
date:       2013-09-13 12:00:00
summary:    While Excel is handy for doing simple stuff with data, eventually there comes a time where you need more.
categories: codemonkey dataphile sensors r
commentIssueId: 12
redirect_from:
  - /codemonkey/dataphile/sensors/r/2013/09/13/leveraging-laboratory-data-with-r/
---

The fog experiment continues, and I've now collected enough data that I can actually start, you know, analyzing it. My first instinct is always to open Excel, and it's definitely a helpful tool for doing some quick visualizations and getting familiar with the way your data is formatted. There comes a point, however, where you have so much data (or it is organized in such a way) that working with it in Excel becomes a handicap. Furthermore, if you're doing repetitive tasks (like, say, pulling load cell measurements of leaves from a [fog simulation experiment]({% post_url 2013-05-23-the-fog-box %}) and normalizing them by [leaf area]({% post_url 2013-07-19-measuring-leaf-area-with-matlab %})) writing a computer program to do the tasks instead of doing them by hand will reduce the likelihood of silly mistakes (or a psychotic break).

I've been on an R binge for the last year, and since I'll need to do some statistical analysis it's the right tool for the job anyway. R makes reading a file stupid easy; the function `readLines()` (note the capital L) will read an entire text file at once and split it into a character vector based on line breaks. You can then use `strsplit()` to split individual or sets of lines into smaller components for more formatting. A this point you can start building dataframes and running calculations 'til the cows come home.

Choosing how to format your data for program reading is not always obvious. In most cases experimental data will require at least a bit of cleanup (it sure does when I run the experiment). The best advice I can give on this front is to (1) minimize any reformatting that must be done by hand, and (2) completely separate preprocessing from analysis. Keeping reformatting to a minimum will reduce the likelihood of errors and make it easy for others to learn the preprocessing steps. One obvious way to do that is to design your template so that the only manipulation of raw data is copying and pasting entire rows. For instance, even if you only need data from the third column, just copy the whole thing. You can have your code deal with subsetting the data, and it also means you won't have to redo anything if you later decide you want the fourth column of data as well. The same goes for keeping the preprocessing and analysis separate: even if it's a really simple calculation that's easy to do while reformatting your data, it's not worth the risk of mistakes. Just let the code take care of it. Of course, this isn't always possible; in my case there are two calculations that still need to be done by hand, and I'm not thrilled about it.

At a minimum, I needed to get specific sets of records from the raw data. These sets of records corresponded to e.g. the baseline voltage, the dry weight of the leaf, and measurements recorded by the LWS sensor at the end of the run. These bits had to be identified manually since the trials were of different lengths and the timing of the measurements varied between runs. However, all trials were guaranteed to have the same data format. This meant that some copying and pasting would be required, but I wouldn't have to code any procedures to try to identify which bits of measurements were to be found in the raw data.

I decided on a format that splits data into chunks according to predefined labels. You can see the format <a href="https://github.com/mkoohafkan/UCBcode-R/blob/master/Fogbox/testdata.txt">here</a>. It's order-independent and easily expandable if additional or optional data  are added in the future; all I need to do is add another label in the file. I actually still use Excel to read the data&#8212;it's much easier to identify the relevant sections in Excel compared to the comma separated format the raw data is recorded in. My <a href="https://github.com/mkoohafkan/UCBcode-R/blob/master/Fogbox/process_fogrun.r">code</a> works by first finding the line numbers that correspond to my predefined labels using `match()`:

{% highlight R %}
# helper function
getmatch = function(tag, dat, stopifmissing=TRUE){
# get indices for data components
  idx = match(tag, dat)
  if(is.na(idx)){
    if(stopifmissing){
      stop(paste(tag, 'is missing.'))
    } else {
      warning(paste(tag, 'is missing.'))
    }
  }
  return(idx)
}
# read lines from file
datalines = readLines(path)
# trial name should be first line
name = datalines[1]
# indices for data processing
startbaseidx = getmatch('STARTING BASE VOLTAGE', datalines) + 1
endbaseidx = getmatch('ENDING BASE VOLTAGE', datalines) + 1
startcalibrationidx = getmatch('STARTING CALIBRATION VOLTAGE', datalines) + 1
endcalibrationidx = getmatch('ENDING CALIBRATION VOLTAGE', datalines, FALSE) + 1
dryleafidx = getmatch('DRY LEAF VOLTAGE', datalines) + 1
wetleafidx = getmatch('WET LEAF VOLTAGE', datalines) + 1
dryragidx = getmatch('DRY RAG VOLTAGE', datalines) + 1
wetragidx = getmatch('WET RAG VOLTAGE', datalines) + 1
startrunidx = getmatch('BEGIN RUN', datalines) + 1
lastLWSidx = getmatch('LAST LWS VOLTAGE', datalines) + 1
maxLWSidx = getmatch('MAX LWS VOLTAGE', datalines) + 1
{% endhighlight %}

If `match()` is unable to find a matching label it returns NA, which I use to check for missing required data (or to warn if optional data is omitted). Then, I sequentially read each line following a label until another label is reached (in this case, any line that starts with a character):

{% highlight R %}
# helper function
scandata = function(dat, idx){
# pull numeric values 
  values = NULL
  n = idx
  while(!is.na(as.numeric(substr(dat[n], 1, 1)))){
    values = rbind(values, unlist(strsplit(dat[n], '\t')))
    n = n + 1
  }
  return(data.frame(timestamp=strptime(values[, 1], "%m/%d/%Y %H:%M:%S"), 
                    record=as.integer(values[, 2]), 
                    LWSmV=as.numeric(values[, 3]),
                    SEmV=as.numeric(values[, 4])))
}
# get start and end base voltages
sbv = scandata(datalines, startbaseidx)
ebv = scandata(datalines, endbaseidx)
# get dry and wet rag voltages
drv = scandata(datalines, dryragidx)
wrv = scandata(datalines, wetragidx)
# get dry and wet leaf voltages
dlv = scandata(datalines, dryleafidx)
wlv = scandata(datalines, wetleafidx)
# get start (and end) calibration voltages
scv = scandata(datalines, startcalibrationidx)
if(!is.na(endcalibrationidx)){ 
  ecv = scandata(datalines, endcalibrationidx)
} else {
  ecv = NA
}
# get LWS voltages
lwsvoltlast = as.numeric(datalines[lastLWSidx])
lwsvoltmax = as.numeric(datalines[maxLWSidx])
# get run data
rundata = scandata(datalines, startrunidx)
runtime = rundata[nrow(rundata), 'timestamp'] - rundata[1, 'timestamp']
mydata = list(name=name, startbasevolt=sbv, endbasevolt=ebv, dryragvolt=drv, 
            wetragvolt=wrv, rundata=rundata, dryleafvolt=dlv, runtime=runtime,
            wetleafvolt=wlv, startcalibvolt=scv, endcalibvolt=ecv,
            lastLWSvolt=lwsvoltlast, maxLWSvolt=lwsvoltmax))
{% endhighlight %}

Now I have all my data in R, coerced into a handy named-list structure and ready for analysis.