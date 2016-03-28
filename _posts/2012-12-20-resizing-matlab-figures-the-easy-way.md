---
layout:     post
title:      Resizing Matlab figures the easy way
date:       2012-12-20 12:00:00
summary:    There are a lot of reasons why you might want to programmatically define the properties of a figure. It turns out you can do this with Matlab pretty easily.
categories: codemonkey matlab visualization
redirect_from:
  - /codemonkey/matlab/visualization/2012/12/20/resizing-matlab-figures-the-easy-way/
---

There are a lot of reasons why you might want to programmatically define the properties of a figure. Perhaps you want to make a slideshow or video out of a series of figures. Maybe you are producing images for a poster, and you have already defined your layout and want your code to output figures you can drop in without having to resize.

It turns out you can do this with [Matlab](http://www.mathworks.com/products/matlab/) pretty easily. The first thing to do is download [export_fig](http://www.mathworks.com/matlabcentral/fileexchange/23629-exportfig) from the [Matlab File Exchange](http://www.mathworks.com/matlabcentral/fileexchange/23629-exportfig). This function allows you to export Matlab figures to a variety of formats that look much more like the figure displayed on-screen compared to what Matlab's `saveas` function provides, and avoids the automatic resizing that would otherwise botch the next step. To actually change the figure size, you just need to set the `position` figure property. The whole process is shown below:

{% highlight matlab %}
fig = figure ; % create a new figure
plot(x, y, 'o') ; % plot some data
% resize the figure window
set(fig, 'units', 'inches', 'position', [10 10 30 20])
% save the figure using export_fig
export_fig('myfigure.eps')
{% endhighlight %}

The `position` vector is defined as `rect = [left bottom width height]`. The first two elements define the distance from the lower-left corner of the screen to the lower-left corner of the full figure window, while the last two elements define the dimensions of the window. I am not convinced that the `units` property actually understands what an inch is, but you can toy around with the values to get your figure to look the way you want; the important thing is that the aspect ratio and figure size are defined consistently. Check out [this help file](http://www.mathworks.com/help/matlab/ref/figure_props.html#Units) for more info on the "units" property.