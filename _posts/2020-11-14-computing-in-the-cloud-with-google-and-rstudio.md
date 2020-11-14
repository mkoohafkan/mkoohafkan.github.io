---
layout:     post
title:      Computing in the Cloud with Google and RStudio
date:       2020-11-14 14:40:00
summary:    I used Google Cloud Platform to do some heavy lifting with R. Here's a quick guide to get started.
categories: codemonkey r google-cloud-platform
commentIssueId: 45
---

I'm chipping away at the last chapter of my dissertation, with the plan
to wrap it up by the end of the year. Part of my last chapter involved 
running tens of thousands of simulations of fish movement in an estuary.
I completed the simulations a while ago and archived the raw output, but
I recently discovered that I needed to reprocess the data for part of my
analysis. Unfortunately, I ended up losing access to my computer
powerhouse due to Covid-19 and having to switch from a desktop computer
sitting in a cubicle to a laptop I could take home with me. The laptop 
isn't particularly powerful, and it locks up trying to process the raw
data. I needed access to more powerful hardware in order to generate 
the outputs I want to present in my dissertation.

Conveniently,
[Google Cloud Platform](https://cloud.google.com/)
is currently offering a free trial of the service.
Signing up now gives you $300 to spend in three months, and they offer
a pretty wide variety of virtual machine (VM) configurations. Even
better, there are a number of R packages that have been built
specifically to work with the Google Cloud Platform. I used two R
packages to complete the work:
[googleComputeEngineR](https://cloud.r-project.org/package=googleComputeEngineR) 
to create the VM and
[googleCloudStorageR](https://cran.r-project.org/package=googleCloudStorageR)
to move data between the virtual machine and the cloud storage 
container or "bucket".


## Setting up a project

The first step is to set up a project. You can do this in the
[Google Cloud Console](https://console.cloud.google.com/). If
you're setting up your Cloud account for the first time this will
probably be one of the initial steps you're asked to complete; if
you already have an existing project, you can create a new one from
the drop-down menu on the blue banner at the top of any console page.
Defining a project lets you create and group VMs and storage buckets.

In order to use googleComputeEngineR and googleCloudStorageR, we also
need to set up a service account, which you can do through the 
[IAM & Admin](https://console.cloud.google.com/iam-admin/serviceaccounts)
console. When you create service account you'll have the option to download
a JSON file containing an authorization key which the R packages use to 
interact with your project.


## Setting up the virtual machine

While you can set up a virtual machine from the 
[Compute Engine](https://console.cloud.google.com/compute/instances)
Console,
it's much easier to create one using GoogleCloudComputeR. The code below
creates an RStudio Server VM using the e2 machine type with 8 GB. Before
creating a VM, make sure to set the project and the authorization file:

```r
library(GoogleCloudComputeR)
sys.setenv("GCE_AUTH_FILE" = "path/to/auth.json")
gce_global_project(project = "my_project_name")


vm2 <- gce_vm(template = "rstudio", zone="us-west2-a",
             name = "my_vm_name",
             username = "my_name", password = "my_password",  
             predefined_type = "e2-highmem-8",
             dynamic_image = "gcr.io/gcer-public/persistent-rstudio",
             disk_size_gb=200)
```

The `dynamic_image` argument lets us specify the RStudio image we want. There are 
[lots of images](https://console.cloud.google.com/gcr/images/gcer-public)
available, but I struggled to get some of them working; the 
"persistent-rstudio" image ended up working best for me.


## Setting up the storage bucket

The next step is to set up a storage bucket. Creating up a bucket is
straightforward in the 
[Storage Console](https://console.cloud.google.com/storage/browser)
(note that there are *lots* of ways to handle storage; the default
Google Cloud Storage Buckets are in the ambiguously-named "Storage"
menu item, which for me was sandwiched betweeen "Memorystore" and 
"Spanner"). The trick here is to use a storage bucket that lives in 
the same "zone" as your VM. If your bucket and VM live in different
zones, the data transfer will be slower and cost more (or possibly
won't work at all). Easy enough to do, but this
is why I had you set up the VM first; since some VM types are
only available in certain regions, you don't want to set up your 
storage bucket only to find out you can't set up the VM you want in
the same region.

The buckets have a drag and drop interface for files so it's really 
easy to upload your stuff through the console. But if you're going 
to need to transfer files from your VM back to the bucket once your 
analysis is completed (which you probably will) then you need to 
need to upload your JSON authorization file to the bucket as well.


## Transferring data to the VM

Getting data from Google Cloud Storage onto the VM is easy. 
After installing the googleCloudStorageR package, just set
the storage bucket and try querying objects:

```r
# install.packages(googleCloudStorageR)
library(googleCloudStorageR)
gcs_global_bucket("rre-paper-bucket")

obj = gcs_list_objects()
```

If you get an error when you try to get a list of objects, it
means that the VM is not able to automatically authenticate
when accessing the bucket. I don't know why this happens (or
rather, I don't know why it is ever able to automatically 
authenticate) but the easiest solution is to create a blank
JSON file on the VM, copy the contents of your authorization
file into the file on the VM, and then set it on the VM using
the command `gcs_auth("path/to/auth-file.json")`.

To transfer files over, I usually do some kind of regex to
filter the object names, and then use a for loop to copy over
each file I need:

```r
search.string = "some regexstring"
selected.obj = obj$name[grep(search.string, obj$name)]

for(o in selected.obj){
  gcs_get_object(o, saveToDisk = o, overwrite = TRUE)
}
```

If you have subfolders in your bucket, you'll probably need
to set the argument `saveToDisk = basename(o)` to drop the
folder path, or recreate the directory structure on the VM.


At this point, your VM is set up and ready to use for your
analysis.

## Transferring data out of the VM

You'll probably want to transfer data or files back out of the 
VM once you're done with your analysis. The googleCloudStorageR
packages makes this really easy too: just use the `gcs_upload()`
function.


```r
out.file = "path/to/output.RData" 
gcs_upload(out.file)
```

As with copying data into the VM, you'll probably need to strip the
folder path by specifying the argument `name = basename(out.file)`
or create the equivalent directory structure in your bucket.

## Don't forget

Once you're done running code on your VM, don't forget to **Shut it down**!
Otherwise it will keep chewing up your money just by being on. You can shut
down your VM from the
[Compute Engine](https://console.cloud.google.com/compute/instances)
Console.


## Other resources

This approach is a pretty barebones, brute-force approach to just 
get stuff done using Google Cloud without taking advantage of
fancy features. But there are quite a few fancy features available!
If you think you're going to be regularly using cloud computing with R,
There's a lot of information on the [cloudyr](http://cloudyr.github.io/)
web site, and a bunch of [packages](http://cloudyr.github.io/packages/index.html)
to help you do more with Google Cloud (and other cloud services!). 
The various packages all have their own help documents, and I suggest you check them
out. In particular, check out the
[googleCloudStorageR "get started"](http://code.markedmondson.me/googleCloudStorageR/articles/googleCloudStorageR.html)
and 
[googleComputeEngineR "setup"](https://cloudyr.github.io/googleComputeEngineR/articles/installation-and-authentication.html)
documents for more info.
