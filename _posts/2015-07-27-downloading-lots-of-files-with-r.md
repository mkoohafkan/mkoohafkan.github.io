---
layout:     post
title:      Downloading lots of files off a (password-protected) website with R
date:       2015-07-27 12:00:00
summary:    Scraping the web is pretty easy with R&#8212;even when accessing a password-protected site.
categories: codemonkey regexpr file-management rcurl r
commentIssueId: 19
---

I've been working on the Russian River Estuary this summer, taking the boat out
to do CTD casts along our regular transect and poring through the data we've 
collected since 2011. One of the things I've been doing is looking through the 
timelapse imagery of the river mouth we've been collecting and trying to 
identify when a closure begins, the idea being that we may see some temporal 
patterns in the evolution of water quality parameters relative to the onset of 
mouth closure. This requires downloading the multitude of timelapse images from 
the [Bodega Ocean Observing Node website](http://bml.ucdavis.edu/boon/), which 
total almost 13 GB. The images are hosted in a password-protected section of 
the site, and the web interface&#8212;while pretty handy for viewing the latest 
images&#8212;doesn't provide any options for batch downloads. 

In an earlier post, I showed how to 
[use R to download files]({% post_url 2014-02-12-downloading-extracting-and-reading-files-in-r %}).
This time, I'm going to show you how to download a bunch of files, and 
(semi)automate getting the list of file URLs to download.

The first thing to do is get a list of URLs for all the files you want to 
download. In my case I am working with the index file at 
`http://bml.ucdavis.edu/boon/dyn/rrmc/archive/2011/`, which looks something
like this: 

{% highlight html %}
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<html>
 <head>
  <title>Index of /boon/dyn/rrmc/archive/2011</title>
 </head>
 <body>
<h1>Index of /boon/dyn/rrmc/archive/2011</h1>
<ul><li><a href="/boon/dyn/rrmc/archive/"> Parent Directory</a></li>
<li><a href="201109300017.jpg"> 201109300017.jpg</a></li>
<li><a href="201110052340.jpg"> 201110052340.jpg</a></li>
<li><a href="201110052350.jpg"> 201110052350.jpg</a></li>
<li><a href="201110060020.jpg"> 201110060020.jpg</a></li>
<li><a href="201110061510.jpg"> 201110061510.jpg</a></li>
<li><a href="201110061610.jpg"> 201110061610.jpg</a></li>
<li><a href="201110061707.jpg"> 201110061707.jpg</a></li>
<li><a href="201110061711.jpg"> 201110061711.jpg</a></li>
</ul>
</body></html>
{% endhighlight %}

Depending on your source file you might be able to do this with 
`download.file`, but since I'm working with a password-protected site I opted
to use the [RCurl](https://cran.r-project.org/web/packages/RCurl/index.html) 
package. I first read the webpage using `getURL`, which returns the entire page
source as a single string; I then use `textConnection` and `readLines` to turn
it into a vector of strings. For security purposes, I use `readline` to prompt
for the username and password instead of including them in plain text in my 
source code.

{% highlight r %}
require(RCurl)
# get login information
baseurl = "http://bml.ucdavis.edu/boon/dyn/rrmc/archive/2011/"
un = readline("Type the username:")
pw = readline("Type the password:")
upw = paste(un, pw, sep = ":")
# get the webpage contents as a single string
webpage = getURL(baseurl, userpwd = upw)
# parse the webpage content into multiple lines
tc = textConnection(webpage)
contents = readLines(tc)
close(tc)
{% endhighlight %}

If this is a one-shot thing for you can skip the next bit of code, as 
it's probably easier to just paste the HTML into Excel and manually pull out 
the links you want. I'm going to show you a more sophisticated solution using 
[regular expressions](https://stat.ethz.ch/R-manual/R-devel/library/base/html/regex.html), 
which are useful if you e.g. need to repeatedly scrape a site or 
download a specific subset of files from a page that contains tons of links. In
my case I want all the jpeg files listed, so I create a regular expression 
to pull just these links out and give me a vector of URLs. I map out the link 
strings using `gregexpr` and extract them using `regmatches`.

{% highlight r %}
# extract href contents of <a> tags that link to .jpg files
rx = gregexpr("(?<=<a href=\")([0-9]{12})(.jpg)(?=\">)",
  contents, perl = TRUE)
urls = unlist(regmatches(contents, rx))
{% endhighlight %}

Let me break down the regular expression 
`(?<=<a href=\")([0-9]{12})(.jpg)(?=\">)` a bit. First, recognize that `()` 
denote groups of characters, so `(.jpg)` searches for the sequence `.jpg` in 
the string. The pattern `([0-9]{12})` says to look for any 12-character 
sequence of digits 0-9; I use this because all of my image names are simply 
numeric datetimes (YYYYMMDDhhmm). The pattern sequence `([0-9]{12})(.jpg)` is
therefore the filename of a given image. The special pattern `(?<=)` is a 
look-ahead assertion; I use the pattern `(?<=<a href=\")` to look for the 
sequence of characters `<a href="` as the start of my search pattern, but I 
don't want it to actually be included in the result. Similarly, the 
pattern `(?=)` is a look-behind assertion, and I use `(?=\">)` to look for 
the sequence `">` as the end of my search pattern without including it in the
result.

Now I have a list of URLs for the files I want to download. Note that they are
all relative links; depending on your needs you may need to do more legwork to
format your URLs properly. In my case I just need to slap 
`http://bml.ucdavis.edu/boon/dyn/rrmc/archive/2011/` in front of them. The next
step is to loop through the URLs and download each file. Because I'm 
downloading images, I use `getBinaryURL` to read the image from the URL and 
`writeBin` to write the contents to a file.

{% highlight r %}
# using a temporary directory
mydir = tempdir()
# generate the full URLS
fileurls = paste0(baseurl, urls)
# generate the destination file paths
filepaths = file.path(mydir, urls)
# create a container for loop outputs
res = vector("list", length = length(filepaths))
# loop through urls and download
for(i in seq(length(fileurls)))
  writeBin(getBinaryURL(fileurls[i], userpwd = upw), 
    con = filepaths[i])
{% endhighlight %}

Since I'm downloading a ton of files, I'll probably want to wrap the loop 
contents in some kind of `tryCatch` statement so that I can go back and check 
if any files failed to download. One solution might be to check the size of the 
result of `getBinaryURL`, since a failure would probably return something 
smaller than the image itself.

That's it! Using these methods I now have over 26,000 images of the river 
mouth. Now if only I could figure out a way to automate the actual analysis...
