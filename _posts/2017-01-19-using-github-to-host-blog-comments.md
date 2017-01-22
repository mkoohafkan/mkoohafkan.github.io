---
layout:     post
title:      Using GitHub to host blog comments&#58; a working example
date:       2017-01-19 19:00:00
summary:    Some smart people figured out how to use the GitHub issue tracker for hosting blog comments, but their example is outdated and missing some steps. Here's how I got it working.
categories: codemonkey jekyll javascript
commentIssueId: 23
---

Up until recently, I was using Disqus as my blog comment system. I never
really liked it much though; it was convenient, but I prefer to have all
of my blog in one place (i.e. on 
[GitHub](https://github.com/mkoohafkan/mkoohafkan.github.io)).
I recently came across 
[this blog](http://ivanzuzak.info/2011/02/18/github-hosted-comments-for-github-hosted-blogs.html), 
which provides instructions on how to query GitHub issues and load them
into a page. It might not be the right solution for everyone---you need
a GitHub account to comment on issues, which some would-be commenters 
might not have---but it seemed perfect for
me. I excitedly copied the code into my repository, disabled 
Disqus, and committed the changes (since I haven't set up Ruby or a 
Jekyll environment on computer, all of my testing happens live). 

Of course, things rarely work out of the box. A quick recheck of the 
blog post revealed it was written back in 2011, and things have, um, 
changed a bit in the last six years. The basic procedure still works, 
but I'll re-explain the process with my updates.

The first two steps are the same: you need to manually create an issue 
in your blog repository's issue tracker for each blog post you publish,
and you need to add the issue number to your post's YAML front matter.
To give you some examples, 
[here](https://github.com/mkoohafkan/mkoohafkan.github.io/issues/23)
is the issue I created for this post, and 
[here](https://github.com/mkoohafkan/mkoohafkan.github.io/blob/master/_posts/2017-01-19-using-github-to-host-blog-comments.md) 
is the markdown file of this post with the issue number included in the 
front matter.

Here's where things start to change. You need to create a liquid 
template for the comments section of your post, which I did by creating
a template  in my `includes` folder. I then added a flag to my post
layout to include the comments template if an issueid is present in the
post front-matter (I also added a site-level flag in case I want to 
disable the system in the future). Within the comments template, you 
need to use Javascript to pull the issue comments using the GitHub API.
However, the API has changed significantly, so I had to modify the 
script to use the right element names. Also, apparently datejs went 
extinct some time ago, so I replaced the datejs code with the standard
Javascript date functionality.

{% highlight javascript %}
<script src="https://ajax.googleapis.com/ajax/libs/jquery/3.1.1/jquery.min.js"></script>
<script type="text/javascript">
  function loadComments(data) {
    for (var i=0; i<data.length; i++) {
      var cuser = data[i].user.login;
      var cuserlink = data[i].user.html_url;
      var clink = data[i].html_url;
      var cbody = data[i].body_html;
      var cavatarlink = data[i].user.avatar_url;      
      var cdate = new Date(data[i].created_at);
      $("#comments").append("<div class='comment'><div class='commentheader'><div class='commentgravatar'>" + '<img src="' + cavatarlink + '" alt="" width="30" height="30">' + "</div><a class='commentuser' href=\""+ cuserlink + "\">" + cuser + "</a><a class='commentdate' href=\"" + clink + "\">" + cdate.toLocaleDateString("en") + " " + cdate.toLocaleTimeString("en") + "</a></div><div class='commentbody'>" + cbody + "</div></div>");
    }
  }
  $.ajax("https://api.github.com/repos/mkoohafkan/mkoohafkan.github.io/issues/{{ page.commentIssueId }}/comments", {
    headers: {Accept: "application/vnd.github.v3.html+json"},
    dataType: "json",
    success: function(msg){
      loadComments(msg);
   }
  });
</script>
{% endhighlight %}

The `headers` specification is important. You can request the comment in
either markdown format or html format depending on the 
[media](https://developer.github.com/v3/media/) type you request. In my 
case, I'm pulling the comments into an html file, so I request the 
`html` media type and extract the `body_html` element.

You can see the template complete with Javascript code 
[here](https://github.com/mkoohafkan/mkoohafkan.github.io/blob/master/_includes/post_comments.html). 
Note that you also need JQuery for this work (as shown in the code 
sample above); my blog already loads JQuery because I use 
[flexslider](https://github.com/woocommerce/FlexSlider), 
so I just modified my head template to load JQuery if either flexslider 
is used or an issue number is specified (in fact, flexslider broke when 
I thought I didn't care about loading JQuery twice). Another 

The other blog then goes into instructions on how enable CORS for a 
GitHub-hosted blog, and how to register and OAuth application for your
blog. I did this when I was still trying to get things working, but you 
don't actually need to do this anymore.

Finally, add some css styling to make your comments look good. The css
provided by the original blog does work right out of the box, but my 
blog template uses SASS so I ported it over to scss and
[modified it a bit](https://github.com/mkoohafkan/mkoohafkan.github.io/blob/master/_sass/_github-comments.scss) 
to keep the look more consistent with my blog. It's still not what I 
want it to be, but I'm pretty terrible at CSS so I haven't really dug 
into it yet.

And you can see the results below! In the future, I'm planning to 
extend this by using comment reactions as a way of moderating which 
comments get published to the blog. However, the 
[reactions API](https://developer.github.com/v3/reactions/)
is still in preview mode, so I'm not going to try anything until it
stabilizes.
