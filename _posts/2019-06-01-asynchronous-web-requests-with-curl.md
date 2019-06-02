---
layout:     post
title:      Asynchronous web requests with R and curl
date:       2019-06-01 10:40:00
summary:    I've been developing web services for our water quality database and implemented asynchronous requests to speed things up.
categories: codemonkey r curl web-services
commentIssueId: 40
---

One of my recent projects at DWR has focused on revamping our data pipelines. We 
have over 60 water quality stations spread across the marsh, all of 
which are telemetered to broadcast continuous water quality to our 
internal database (and also to [CDEC](http://cdec.water.ca.gov/)) 
and all of which require regular maintenance, calibration, and 
quality control and assurance. We recently adopted a more 
[rigorous QAQC process](https://pubs.usgs.gov/tm/2006/tm1D3/pdf/TM1D3.pdf)
which requires our technicians to enter **a lot** more field visit data 
into our database and perform more complex calculations to determine quality
ratings (in addition to the manual inspections they were already doing).
I've been working to streamline our data pipeline and reduce the amount
of data transcription and manual calculations our technicians need to
do. The pipeline I've developed consists of a few steps, starting with
moving our field visit data entry to electronic data sheets (using Excel,
since that is what our technicians are familiar with) and using our
agency's OneDrive to allow offline data entry when in the field
and automatic uploading to our servers when they get back to the office.
Most of my efforts have focused on what happens *after* that, and my
pipeline for writing the field data to the database and calculating the 
QAQC ratings relies on new
[web services](https://stackoverflow.com/a/226159) functionality 
for our database that I've been implementing and testing. These web 
services allow me to query, insert, and update data programmatically 
(i.e. with R). 

The web services on the server-side are implemented in C#, but my data 
pipeline is mostly in R. The 
[`curl`](https://cran.r-project.org/package=curl) package provides a 
lot of neat functionality for making web requests. A lot of R users 
recommend [`httr`](https://cran.r-project.org/package=httr) for this,
but `curl` provides one important feature that `httr` does not (yet)
support: the ability to make 
[asynchronous web requests](https://docs.oracle.com/cd/E15523_01/web.1111/e15184/asynch.htm#WSCPT137).
Using asynchronous requests allows me to do multiple web requests 
simultaneously, which significantly speeds up my process.

Making asynchronous requests with `curl` isn't much harder than making
synchronous requests. Compare the following two functions for 
performing 
[GET requests](https://www.w3schools.com/tags/ref_httpmethods.asp):

```r
# synchronous request - single URL
basic_get = function(service.url) {
  # perform the request
  result = curl_fetch_memory(service.url, handle = wqpr_handle())
  # process the result
  rawToChar(result$content)
}}
```

```r
# asynchronous request - vector of URLs
multi_get = function(service.urls) {
  # container to store the results
  results = list()
  # callback function to capture the result of each request
  cb = function(res) {
    results <<- append(results, list(res))
  }
  # create the request pool
  pool = new_pool()
  # add the requests to the pool
  lapply(service.urls, curl_fetch_multi, pool = pool, done = cb,
    fail = cb, handle = new_handle())
  # make the requests
  out = multi_run(pool = pool)
  # process the results
  lapply(results, function(x) rawToChar(x$content))
}
```

These functions are pretty barebones (I removed the error handling code
for simplicity) but they demonstrate the difference between the two 
approaches. The most important thing to note here is that when 
requests are performed asynchronously, it's very likely that the requests
will complete *in a different order than you provided*. This is usually 
fine with a GET request, since the information you retrieve will 
(probably) have some identifying information that you can use to link
it back to the original request. However, can be a serious issue with
POST requests, because the data you are sending out is contained in the
request body and web service response might not contain *ANY*
identifying information. So if you make 5 POST requests and two of them
fail, how can you determine *which* requests failed?

My solution to this takes advantage of R' as a [functional 
programming language](http://adv-r.had.co.nz/Functional-programming.html#functional-programming).
Instead of creating a single callback function like above, I create a 
unique callback function *for each request* that captures information 
that would not otherwise be returned by the web service request. In the 
case below, each callback function assigns a specific name to the request,
which can then be matched back to the originating request.

```r
# asynchronous request - with identifiers
multi_get2 = function(service.urls) {
  # container to store the results
  results = list()
  # callback function generator - returns a callback function with ID
  cb_gen = function(id) {
    function(res) {
      # assign names to added results
      results[[id]] <<- res
    }
  }
  # define the IDs
  ids = paste0("request_", seq_along(service.urls))
  define the callback functions
  cbs = lapply(ids, cb_gen)
  # create the request pool
  pool = new_pool()
  # add the requests to the pool - use specific callback function for 
  # each request
  lapply(seq_along(service.urls), function(i)
    curl_fetch_multi(service.urls[i], pool = pool,
      done = cbs[[i]], fail = cbs[[i]],
      handle = new_handle())	
  # make the requests
  out = multi_run(pool = pool)
  # process the results in the same order that the URLs were given
  lapply(results[ids], function(x) rawToChar(x$content))
}
```

And that's it! By using the IDs I generated to order the list of 
results, I'm guaranteed to get the output of my requests in the same
order as the service URLs I provided, making it trivial to identify the
response (e.g. success or failure) of each request. 

Implementing asynchronous requests **massively** sped up my pipeline, 
and I'm getting ready to move the web services over to our production
database. The next question is---can I convince our technicians to 
adopt the new process?
