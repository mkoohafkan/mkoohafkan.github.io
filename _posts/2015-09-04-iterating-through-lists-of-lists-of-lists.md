---
layout:     post
title:      Iterating through lists of lists (of lists of lists of...) with R
date:       2015-09-04 12:00:00
summary:    When you're doing big data analysis with R, you're probably working with nested lists. Here's one way to climb back up the rabbit hole.
categories: codemonkey file-management recursion r
commentIssueId: 20
---

A friend of mine has been doing a lot of work analyzing hydrologic time series 
from a large number of stream gauges in California. She's been doing all kinds 
of statistical analyses with various transformations and combinations of the 
dataset, which has inevitably led to working with nested lists of horrifying 
proportions. While nested lists are a pretty handy structure for programming, 
they lose their appeal pretty quickly once it's time for a human to start 
accessing their contents.

She approached me yesterday with what sounds like a simple question:

> "How can I loop through a nested list of dataframes and write each dataframe 
> element to a directory structure that matches the list structure?"

In other words, if she has a list of three sublists, and each sublist 
contains two dataframes, she wants to end up with a folder that contains three 
folders, each of which contains two CSV files. For a normal list, this would be 
a trivial application of `lapply` and `write.csv`. For a nested list it gets 
trickier, but if you know the structure beforehand it's just a matter of 
writing some nested loops. But what if you don't know the list structure 
beforehand? And what if some of these (sub)lists also contain elements other 
than dataframes that you want to ignore?

My solution consists of three steps: first, I generate a flat list of element 
identifiers by sending a recursive function to `lapply` that only returns a 
value for elements in the nested list that are dataframes. By default, `lapply` 
uses the list names for the returned object; for a nested list I end 
up with names like "foo.A.one", "foo.A.two", "foo.B.one", etc. Then, I convert 
the list of identifiers to a list of 
target filepaths by using `gsub` to replace "." with "/". I can then extract 
folder names from the list of filepaths using `dirname` and create the 
directory structure via `dir.create`. Finally, I convert the element 
identifiers to a list of expressions that evaluate `write.csv` for each list 
element, and use `eval` and `lapply` to write the files to the appropriate 
folders.

At the request of my friend, I wrapped this methodology into the poorly-named 
function `magicCSV`, with an additional argument that lets you specify the 
top-level directory to write the directory structure and files to.

```R
magicCSV = function(startlist, targetdir){
  ff = function(x){ 
    if (class(x) == "list") 
      lapply(x, ff) 
    else if(class(x) == "data.frame") 
      TRUE
    else
      NULL
  }
  lnames = names(unlist(lapply(startlist, ff)))
  fnames = file.path(file.path(targetdir), paste0(gsub(".", "/", 
    lnames, fixed = TRUE), ".csv"))
  dirnames = unlist(lapply(fnames, dirname))
  varnames = paste0("startlist$", gsub(".", "$", lnames, 
    fixed = TRUE))
  evalstrings = paste0('write.csv(', varnames, ', file ="', 
    fnames, '")')
  exprs = lapply(evalstrings, function(x) parse(text = gsub("\\", 
    "/", x, fixed = TRUE)))

  suppressWarnings(lapply(dirnames, dir.create, recursive = TRUE))

  invisible(lapply(exprs, eval, envir = environment()))
}
```

Note the use of `file.path` to ensure that it doesn't matter if a user specifies
a target directory as `C:/foo` or `C:/foo/`, and the weirdness of replacing 
`\\` with `/` when creating the expressions so that `eval` doesn't break (I 
don't know why we have to do this). While it is ordinarily better to use 
`Recall` to call functions recursively, we can't do that here because `Recall`
doesn't work when passed as a function argument (such as to `lapply`).

Here's some code you can use to test out `magicCSV`:

```R
# function to create an arbitrary dataframe
rdf = function(){ 
  data.frame(replicate(sample(seq(2, 6), 1), rnorm(10))) 
}
# make a nested list of dataframes for testing
test = list(
  foo = list(
    A = rdf(), 
    B = rdf(), 
    roo = factor(sample(LETTERS, 100, replace = TRUE))),
  bar = list(
    C = rdf(), 
    D = rdf()),
  baz = list(
    boo = list(
      E = rdf(), 
      F = rdf()),
    far = list(
      gar = list(
        G = rdf(), 
        H = rdf()),
      gaz = list(
        I = rdf(), 
        J = rdf()),
      gab = factor(sample(LETTERS, 100, replace = TRUE))
    )
  )
)

# create a temporary directory
mydir = tempdir()
# test it out
magicCSV(test, mydir)
dir(mydir, recursive = TRUE)
read.csv(file.path(mydir, "bar", "C.csv"), row.names = 1)
```

And there you have it! It would be pretty easy to generalize this function 
further by e.g. passing a function to allow the user to create a custom `eval` 
statement, or an argument specifying what class to search the nested loop for. 
I'll leave that to you.

**EDIT**

It turns out there's a better way to do the above that avoids the messiness 
of `eval` and `parse`. The `[[]]` bracketing is actually way more powerful than
I originally realized; you can actually access nested lists by using vectors!
Below is a slightly rewritten version of `magicCSV` that takes advantage of this.

```R
magicCSV = function(startlist, targetdir){
  ff = function(x){ 
    if (class(x) == "list") 
      lapply(x, ff) 
    else if(class(x) == "data.frame") 
      TRUE
    else
      NULL
  }
  lnames = names(unlist(lapply(startlist, ff)))

  fnames = file.path(file.path(targetdir), paste0(gsub(".", "/", 
    lnames, fixed = TRUE), ".csv"))
  dirnames = unlist(lapply(fnames, dirname))

  varnames = strsplit(lnames, split = ".", fixed = TRUE)
  
  suppressWarnings(lapply(dirnames, dir.create, recursive = TRUE))
  invisible(lapply(seq_along(varnames), function(i) 
    write.csv(startlist[[varnames[[i]]]], file = fnames[[i]])))
}
```

That's much better!