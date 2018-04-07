---
layout:     post
title:      Muddling through Docker and Google Earth Engine
date:       2017-10-09 20:00:00
summary:    I bumbled my way through building a custom Docker image for Google Earth Engine that lets me load additional Python modules. 
categories: codemonkey google-earth-engine docker python
commentIssueId: 30
---

I recently joined the 
[Google Earth Engine](https://earthengine.google.com/) 
[developer community](https://developers.google.com/earth-engine/).
I have been successfully using a new method I developed (publication 
pending!) for exploring remotely-sensed data of emergent sandbars,
and I want to explore its applicability to other systems. Google 
Earth Engine's massive data catalog and Python interface is an 
obvious choice of test environment, and it's pretty easy to get
started. 

The developer interface is provided via a 
[Docker](https://www.docker.com/) 
container, and it's fairly straightforward to 
[run the container locally](https://developers.google.com/earth-engine/python_install),
which lets you run stuff offline and avoid cloud computing costs.
Getting the standard docker container up and running is fairly simple,
provided you have access to hardware virtualization. In my case, I
upgraded my Windows 10 Home edition to 
[Education edition](https://www.microsoft.com/en-us/education/Products/Windows/default.aspx), 
which I was able to get for free from my university. It was then a 
simple matter of enabling hardware virtualization in my system BIOS
and 
[installing Docker Community Edition](https://store.docker.com/editions/community/docker-ce-desktop-windows).
Simply copying and pasting Google's 
[instructions](https://developers.google.com/earth-engine/python_install-datalab-local) 
got me up and running (once I realized that I needed to run the 
commands in *terminal*, not PowerShell!).

The method I want to test out relies on some 
modules that aren't part of the standard Python distribution, 
and they aren't included in the GEE container. However, it's not
straightforward (at least to me) to install additional stuff in an
existing container. In general, it's better to build a custom 
container that installs any extra stuff from the get-go (this makes 
your container more portable).

Despite never having used Docker before, I managed to build a custom 
container based on the GEE image (with some 
[assistance](https://stackoverflow.com/questions/45656276/installing-python-modules-on-the-local-google-earth-engine-docker-image) 
from a friendly StackOverflow user). I've written out instructions
below, but honestly I don't really know what all the steps mean.

  1. Follow Google's 
     [instructions](https://developers.google.com/earth-engine/python_install-datalab-local).
     to get a feel getting GEE set up locally. You should end up with 
     a folder "datalab-ee" in your workspace (for me, it's in 
     `C:\Users\michael\workspace`).

  2. Create a new folder in your workspace. I named my container 
     `datalab-mk`, so I made a folder 
     `C:\Users\michael\workspace\datalab-mk`.

  3. Create a file `Dockerfile` (no extension) in the `datalab-mk`
     folder with the following contents:
     ```
     FROM gcr.io/earthengine-project/datalab-ee:latest
     WORKDIR C:/Users/michael/workspace/datalab-ee-mk
     COPY . .
     RUN pip install -r requirements.txt
     ```
     The GEE image includes [pip](https://pypi.python.org/pypi/pip), 
     so last line lets you install additional Python modules in the
     container. I *think* the `COPY` command will copy any additional
     files you placed in the container folder into the container 
     environment.

  4. Create a file `requirements.txt` in the `datalab-mk` folder.
     This file contains the list of Python modules to install. For
     example, my `requirements.txt` file contains the following line:
     ```
     networkx==1.11
     ```
     which installs the Python module `networkx` (version 1.11).

  5. Run the new container with this slight modification to 
     the [workspace setup](https://developers.google.com/earth-engine/python_install-datalab-local#step-2---define-a-local-workspace) 
     and [container creation](https://developers.google.com/earth-engine/python_install-datalab-local#step-3---create-a-container) 
     instructions (note the references to `datalab-mk`):
     ```
     set "GCP_PROJECT_ID=YOUR_PROJECT_ID"
     set "CONTAINER_IMAGE_NAME=datalab:mk"
     set "HOME=%HOMEDRIVE%%HOMEPATH%"
     set "WORKSPACE=%HOME%\workspace\datalab-ee-mk"
     mkdir "%WORKSPACE%"
     cd %WORKSPACE%

     docker run -it -p "127.0.0.1:8081:8080" -v "%WORKSPACE%:/content" -e "PROJECT_ID=%GCP_PROJECT_ID%" %CONTAINER_IMAGE_NAME%
     ```
  6. Follow the [authentication](https://developers.google.com/earth-engine/python_install-datalab-local#authenticating-to-earth-engine) 
     instructions as normal.

And that's it! Now you have a GEE container with an additional Python
module. At this point, you can further customize the 
[Dockerfile](https://docs.docker.com/engine/reference/builder/)
to set up additional software or do other fancy stuff. There is some
pretty decent information on 
[best practices for Dockerfiles](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/)
that I'll be making heavy use of. But I think I'm over the hurdle and
will be able to add the customizations I need to apply my methods to
GEE data.
