---
layout:     post
title:      Synchronizing video and data with R and Matlab, part 1&#58; abusing ggplot
date:       2013-12-28 12:00:00
summary:    Here's one strategy for creating animated plots that uses both R and Matlab. I'll show you how to use ggplot2 to create the animation frames.
categories: codemonkey animation  ggplot2 r
commentIssueId: 14
redirect_from:
  - /codemonkey/animation/ggplot2/r/2013/12/28/synchronizing-video-and-data-with-r-and-matlab-part-1/
---
I participated in a poster session at the AGU 2013 conference in San Francisco. I presented on a fog monitoring study I've been conducting in collaboration with the San Francisco Public Utilities Commission, and as part of the study we have recorded 5-minute time-lapse imagery in conjunction with some soil moisture and leaf wetness sensor data. I'd seen some poster presentations bring laptops to show videos at AGU the previous year, and I decided to do the same. My goal was to create a video from the timelapse imagery that included a 'moving plot' of my sensor data that was synchronized with the time-lapse images. In this post I'll show you how to create a series of plots using R's `ggplot2` package which could be used as frames for a moving plot video. Next time, I'll go over how to work with videos in Matlab and merge the plot images with each video frame.

Let's say you want to make a time-lapse plot of soil moisture measured at some location. For simplicity, assume the data consists of a csv file with two columns: 'timestamp' and 'soil.moisture'. You can easily make a series of plots by creating a series of line aesthetics:

{% highlight r %}
mydata = read.csv('myfile.csv')
names(mydata) = c('timestamp', 'soil.moisture')
# define base plot
p = ggplot(NULL, aes(x=timestamp, y=soil.moisture))
# create plot for each row of data
for(i in seq(nrow(mydata))){
  # create plot frame
  np = p + geom_line(data=mydata[seq(i),])
  # save plot to numbered image
  ggsave(sprintf('plot%d.tiff', i), plot=np, width=6, height=4, dpi=200)
}
{% endhighlight %}

This procedure is fine for creating a series of simple plots. But what if you wanted to make a time-lapse plot of soil moisture measured at two different locations in a soil column? Perhaps the data is organized into three columns: 'timestamp', 'surface.moisture', and 'depth.moisture'. In order to use `ggplot` the proper way, you would first melt the data so that you could use ggplot aesthetics:

{% highlight r %}
mydata = read.csv('myfile.csv')
names(mydata) = c('timestamp', 'surface.moisture', 'depth.moisture')
melt_data = function(d){
  md = melt(d, id.vars='timestamp', measure.vars=c('surface.moisture', 'depth.moisture'))
  names(md) = c('timestamp', 'location', 'measurement')
  return(md)
}
ggplot(melt_data(mydata), aes(x=timestamp, y=measurement, color=location) + geom_line()
{% endhighlight %}
However, if we want to use the strategy above to make a sequence of plots, we have to melt the data at every interval:
{% highlight r %}
baseplot = ggplot(NULL, aes(x=timestamp, y=measurement, color=location))
for(i in seq(nrow(thedata)){
  np = baseplot + geom_line(data=melt_data(mydata[seq(i),]))
  print(np)
}
{% endhighlight %}

You can speed that up a bit by passing `data[seq(i-1, i),]` to `melt_data()` instead of `data[seq(i),]` and successively adding each layer to the same plot object, but that still takes a long time compared to the first example. So let's abuse `ggplot2` and force it to do things like it does in the first example:

{% highlight r %}
baseplot = ggplot(NULL, aes(x=timestamp))
for(i in seq(nrow(thedata)){
  np = baseplot + geom_line(data=mydata[seq(i),], mapping=aes(y=surface.moisture, color='surface')) 
       + geom_line(data=mydata[seq(i),], mapping=aes(y=depth.moisture, color='depth')) 
  print(np)
}
{% endhighlight %}

Notice that the color aesthetics are explicitly defined using strings, rather than melting the data and using an identifier column for coloring. This method is considerably faster than melting the data each step, and the plots look just as good as if you used the `ggplot2` aesthetics the proper way. Now, you have a method for creating a sequence of plot images. I'll show you how to pull these images together to create a video in a later post.