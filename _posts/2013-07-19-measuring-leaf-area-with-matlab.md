---
layout:     post
title:      Measuring leaf area with Matlab
date:       2013-07-19 12:00:00
summary:    Measuring the surface area of a leaf is quite simple with Matlab.
categories: codemonkey image-analysis Matlab
redirect_from:
  - /codemonkey/image-analysis/matlab/2013/07/19/measuring-leaf-area-with-matlab/
---

I mentioned earlier that I've been conducting [laboratory experiments regarding fog deposition]({% post_url 2013-05-23-the-fog-box %}). One thing I want to do is present my findings in terms of water mass deposited per unit leaf area, which means I have to go to the trouble of measuring the area of each leaf. One way of doing this is through image analysis: we use a picture of a leaf along with a picture of some known reference area to calculate the area of the leaf. The procedure outlined below is partly based on the work of an undergraduate who briefly joined the Thompson Lab to gain some research experience.

The procedure is really quite simple: we use a black and white scanner to scan the leaf, and place a reference area (a printout of a solid-colored 1 inch square) next to the leaf so that we can measure area independently of scanner resolution. As an example, here's a recent scan of a black oak leaf:

![scan of a black oak leaf](/images/2013-07-19-scanned-leaf.png)

Getting started with an image in Matlab is pretty easy; just use `imread()` and `imshow()` to load and display images, respectively. Black and white (BW) images are represented in Matlab as Boolean arrays, with each element corresponding to a pixel. Black pixels have a value of 0 (false) while white pixels have a value of 1 (true). Personally that seems counter-intuitive to me, so I often use `imcomplement()` to flip the image.

As you can see from the image above, we need to do a bit of work before we can start calculating area. The different 'shades' of grey are really just varying proportions of black and white pixels, and we need to 'fill in' these areas to get a solid shape. We also want to remove specs and artifacts (like the shadows on the edges of the scan) so that we're measuring the area of our shapes, but not dirt on the scanner glass. I use `imopen()` and `imclose()` to<a href="http://www.dspguide.com/ch25/4.htm"> morphologically open and close</a> the image a few times and clean everything up. Once that's done, we can easily calculate area by counting the number of leaf pixels (using `bwarea()`) and comparing that to the number of pixels in our reference scale. Something like this:

{% highlight matlab %}
function [realarea, leaf, scale, original] = getleafarea(imgfolder, imgname)
    % assumes scale is in lower right corner, leaf is on top half of image
    original=imread([imgfolder '/' imgname]) ;
    leaf = original(5:end-500, 5:end-5, :, :) ;
    scale = original(end-500:end-5, end-500:end-5, :) ;
    % processing scale
    scale = imclose(scale, strel('square', 2)) ;
    scale = imopen(scale, strel('square', 2)) ;
    scale = imclose(scale, strel('line', 100, 0)) ;
    scale = imclose(scale, strel('line', 100, 90)) ;
    % processing leaf
    leaf = imopen(leaf, strel('disk', 1)) ;
    leaf = imclose(leaf, strel('disk', 3)) ;
    % write processed images to file for verification
    imwrite(scale, [imgfolder '/processed_scale-' imgname]) ;
    imwrite(leaf, [imgfolder '/processed_leaf-' imgname]) ;
    % calculate area
    % assumes scale is 1-in square
    leafarea = bwarea(imcomplement(leaf)) ;
    scalearea = bwarea(imcomplement(scale)) ;
    realarea = leafarea/scalearea ; % in inches
end
{% endhighlight %}

And here's the result:

![black oak leaf image after processing](/images/2013-07-19-processed-leaf.png)

Neat huh? Note that I've lost the spine-like tips on the leaf, but that's all right because they don't really contribute to the leaf area on which fog condenses. The procedure works just fine for other types of leaves as well, unless they happen to be so thin that the scanner light shines through them. In those cases, I'll probably have to take a marker to the leaves to darken them.