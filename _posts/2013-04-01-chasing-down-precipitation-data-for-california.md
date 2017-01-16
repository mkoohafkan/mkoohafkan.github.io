---
layout:     post
title:      Chasing down precipitation data for California
date:       2013-04-01 12:00:00
summary:    Precipitation data for California is scattered across a number of different services.
categories: dataphile gis
commentIssueId: 7
redirect_from:
  - /dataphile/gis/2013/04/01/chasing-down-precipitation-data-for-california/
---

I recently added a new project to my research, which involves investigating seasonal fragmentation of streams. My advisor has developed some interesting theory based on <a href="http://onlinelibrary.wiley.com/doi/10.1029/2006WR005043/abstract">Botter et al's probabilistic characterization of hydrologic flows</a>, and in order to do some basic validation I needed to obtain daily precipitation and streamflow data for about ten undammed catchments.

The USGS <a href="http://waterdata.usgs.gov/nwis">National Water Information System</a> is great---for stream gauge data, at least. Thom Moran (another researcher at Berkeley) has also developed <a href="http://www.ocf.berkeley.edu/~tcmoran/CA_Catchments_html/catchments_overview_RefCatch1.html">his own interface</a> for looking at watershed data from the USGS and from other sources. Identifying undammed watersheds turned out to be a simple matter of looking through the USGS "reference" catchments and picking out some good tributaries (whitewater rafting websites were surprisingly helpful as well). The USGS precipitation data, however, leaves much to be desired (at least for California). In fact, precipitation data for California is scattered between a number of networks: The <a href="http://www.raws.dri.edu/">RAWS climate archive</a>, the <a href="http://cdec.water.ca.gov/">California Data Exchange Center</a> and the <a href="http://www.nws.noaa.gov/om/coop/">NWS Cooperative Observer Program</a>, to name a few. I'm sure there is more data at the county and local levels as well, but there doesn't seem to be any service that aggregates this data.

My needs boiled down to a simple GIS analysis: I had to identify precipitation gauges near the stream gauges I had picked out. Unfortunately, switching back and forth between multiple data sites (and variably-terrible map interfaces) to try to find appropriate weather stations turned out to be a huge pain. It didn't take me long to realize I'd be better off making my own GIS layer with the locations of weather stations in California. Once I was able look at all the nearby gauges at once, I could pick out a short list of gauges and check promising stations individually for data availability etc.

It turns out you can get custom lists of <a href="http://cdec.water.ca.gov/cgi-progs/staSearch">CDEC station locations</a> pretty easily, and you have lots of filtering options. RAWS is a little trickier, as you only have very <a href="http://www.wrcc.dri.edu/cgi-bin/raws1_pl">rudimentary filtering options</a> (although filtering by state was enough for me). More problematic is that the resulting page shows stations in a drop-down list, meaning you can't copy and paste the info. Not to be deterred, I looked at the page source and eureka! all of the information in the drop down list also shows up in the HTML code. Can't stop me from copying and pasting that!

With a little bit of Excel and this <a href="http://www.earthpoint.us/ExcelToKml.aspx">handy website</a> I was able to generate a Google Earth kml file containing the names and locations of all the CDEC and RAWS gauges. Now I can zoom to individual stream gauges in Google Earth and quickly spot promising weather stations nearby. I'll update this post if find the station locations for the COOP network; in the meantime, you can check them out on <a href="http://gis.ncdc.noaa.gov/map/viewer/">NOAA's online map interface</a>.
