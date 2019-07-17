---
layout:     post
title:      Synchronizing video and data with R and Matlab, part 2&#58; video I&#47;O with Matlab
date:       2014-01-10 12:00:00
summary:    Here's one strategy for creating animated plots that uses both R and Matlab. I'll go over how to work with videos in Matlab and merge images for use as video frames.
categories: codemonkey animation matlab
commentIssueId: 15
redirect_from:
  - /codemonkey/animation/matlab/2014/01/10/synchronizing-video-and-data-with-r-and-matlab-part-2/
---

I participated in a poster session at the AGU 2013 conference in San Francisco. I presented on a fog monitoring study I've been conducting in collaboration with the San Francisco Public Utilities Commission, and as part of the study we recorded 5-minute time-lapse imagery in conjunction with some soil moisture and leaf wetness sensor data. I'd seen some poster presentations bring laptops to show videos at AGU the previous year, and I decided to do the same. My goal was to create a video from the timelapse imagery that included a 'moving plot' of my sensor data that was synchronized with the time-lapse images. In an earlier post, I showed how to [create a series of plots]({% post_url 2013-12-28-synchronizing-video-and-data-with-r-and-matlab-part-1 %}) using `ggplot2` which could be used as frames for a moving plot video. In this post, I'll go over how to work with videos in Matlab and merge the plot images with each video frame.

The decision to work with Matlab was pragmatic. While it would have been better in the long run to use Python's <a href="http://opencv.org/">openCV</a> module, I was already somewhat familiar with Matlab's video processing capabilities. I also found this <a href="http://www.mathworks.com/help/matlab/examples/convert-between-image-sequences-and-video.html">absolutely fantastic tutorial</a> which shows you how to read and write videos from images step by step, with some extra tricks for working with windows directories. This meant that I could create my process in a matter of hours, rather than days if I were teaching myself openCV from scratch.

The first step is to trim the timelapse video to the range of the data. In my case, this was relatively easy as my sensors are operating at the same interval as time-lapse imagery (5 minutes). Therefore, as long as there are no breaks in the sensor data or timelapse imagery I can simply pick a window of time when both the sensors and the images are recording. The times of the video and the sensor readings probably won't line up exactly (e.g. images start at 11:05 but the sensor starts at 11:07), but that's fine for my needs. The first step is to trim the video down to the appropriate window. In my case, the timelapse video has a timestamp on the bottom. So I'll simply extract all the timestamps to images, and name them by their frame position:

{% highlight matlab %}
% load the input video
invideo = VideoReader(videoname) ;
% get the number of frames
nframes = invideo.NumberOfFrames ; 
% create folder to house exported frames
outfolder = [videoname '_timestamps'] ;
mkdir(outfolder) ;
% extract the frames
for i = 1:nframes
  % extract frame as RGB image 
  img = read(inobj, i) ;
  % write frame to a file. 
  % timestamp is 705px to 720px, across width of frame
  f = [outfolder '\' videoname '_timestamps' sprintf('_%d.',i) imgformat] ;
imwrite(img(705:720, :, :), f) ;
{% endhighlight %}

This creates a folder with the timestamps as images, named with their frame index number. I then just have to find the start and end frames corresponding to my time frame and note their index numbers.

The next step is to load the plot images. Assuming they are numbered sequentially, you can use regular expressions to load them all:

{% highlight matlab %}
% assuming folder contains only the plot images as tiff files, e.g. 'plot1.tiff'
matchstring = '*tiff' ;
% get list of files with specified extension
filelist = ls(matchstring) ;
imagenames = strtrim(mat2cell(filelist, ones(size(filelist, 1), 1))) ;
% sort files
imagestrings = regexp([imagenames{:}], '(\d*)', 'match') ;
imagenumbers = str2double(imagestrings) ;
[~,sortidx] = sort(imagenumbers) ;
sortednames = imagenames(sortidx) ;
{% endhighlight %}

`sortednames` is a cell array containing the filenames or paths of the plot images, in ascending order. Next, we create the output video file:
{% highlight matlab %}outobj = VideoWriter(outvideo, 'MPEG-4') ;
outobj.FrameRate = 4 ;{% endhighlight %}
I should say here that profiles in Matlab are pretty important. you can type `VideoWriter.getProfiles` to see a list of available profiles on your machine. Making sure a video can play outside of Matlab means you need to be careful about what video profile you choose and what resolution your output frames are. In my case, the cameras I was using record in 720p, so I simply expanded to 1080p and created plot images with the correct resolution to simply stick on the bottom of each timelapse frame. This also means that I can output in .mp4 format, which keeps the filesize manageable. To write the video, I simply loop through each frame of the video in my specified range, and merge it with the plot image. Since images in Matlab are loaded as a `NxMx3` matrix (or `NxMx4` if the image supports transparency), I can simply use `vertcat()` to concatenate the image matrices and create the new frame:

{% highlight matlab %}
% open the output video for writing
open(outobj)
% assume I want to start at the 10th frame of the video, and end at the 100th frame
for i = 10:100 % frame indexes
  % expand the video frame to 1080p
  frame = padarray(read(inobj, i), [0 320]) ; % only pad the sides
  % load the next plot image, note the offset
  img = imread(sortednames{i - 10 + 1}) ;
  % concatenate the matrices
  newframe = vertcat(frame, img) ;
  % write to the new video
  outobj.writeVideo(newframe) ;
end
% finalize the output video
close(outobj) ;
{% endhighlight %}

And you can end up with something like this:
<iframe src="//player.vimeo.com/video/82855306" width="500" height="283" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe> <p><a href="http://vimeo.com/82855306">outvideo.avi</a> from <a href="http://vimeo.com/user23821232">michael koohafkan</a> on <a href="https://vimeo.com">Vimeo</a>.</p>
The procedure I outlined here isn't exactly what I did; I ended up creating a number of general functions for manipulating videos in Matlab. You can check them out <a href="https://github.com/mkoohafkan/UCBcode-Matlab/tree/master/VideoProcessing">here</a>.