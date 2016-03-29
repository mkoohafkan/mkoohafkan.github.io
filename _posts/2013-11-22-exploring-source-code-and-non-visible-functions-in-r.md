---
layout:     post
title:      Exploring source code and non-visible functions in R
date:       2013-11-22 12:00:00
summary:    Viewing an R function's source code is usually straightforward, but sometimes a few extra steps are needed.
categories: codemonkey r
redirect_from:
  - /codemonkey/r/2013/11/22/exploring-source-code-and-non-visible-functions-in-r/
---

I've been playing around with the `intamap` package recently as I'm interested in how it implements inverse distance weighting interpolation. I heard that the package uses a cross-validation approach to optimize the distance power, but I want to understand how the algorithm works and that means looking at the source code. Most of the time, viewing a function's source code is straightforward: just type the name of the function in the R prompt, without parentheses. For instance:

{% highlight R %}
> require(intamap)
> interpolate
{% endhighlight %}

gives you the source code for the interpolate function. But sometimes typing the function name doesn't seem to give you what you want:

{% highlight R %}
> require(intamap)
> spatialPredict
function (object, ...) 
UseMethod("spatialPredict")
<environment: namespace:intamap>
{% endhighlight %}

If the function or class is overloaded (defined for multiple types of inputs), you need an additional step to view the code for the specific version of the function you're interested in. You can get a list of the versions of the function using `methods()` for S3 classes or `showMethods()` for S4 classes (if you're interested in learning more about these classes, <a href="http://stackoverflow.com/questions/6450803/class-in-r-s3-vs-s4">this StackOverflow link</a> will get you started). Once you know which version of the function you want to look at, just type it's full name to view the source code:

{% highlight R %}
> methods(mean)
[1] mean.Date    mean.default    mean.difftime
[4] mean.POSIXct    mean.POSIXlt    mean.yearmon*
[7] mean.yearqtr*    mean.zoo*
> mean.default
{% endhighlight %}

For some functions, `methods()` will return a list in which some or all of the entries end with an asterisk (*). Trying to view the source code by just typing the function name yields an error:

{% highlight R %}
> require(intamap)
> methods(spatialPredict)
[1] spatialPredict.automap*    spatialPredict.copula*         
[3] spatialPredict.default*    spatialPredict.idw*            
[5] spatialPredict.linearVariogram*
[6] spatialPredict.transGaussian*  
[7] spatialPredict.yamamoto*  
> spatialPredict.idw
Error: object 'spatialPredict.idw' not found
{% endhighlight %}

The reason for this is that functions marked with an asterisk are non-visible. Meaning, they aren't defined in the global environment. That doesn't mean you can't view the source code though, you just need to specify the namespace as well:

{% highlight R %}
> intamap:::spatialPredict.idw
{% endhighlight %}

Now you have the basic tools you need to explore R packages and check the source code for functions you're interested in.