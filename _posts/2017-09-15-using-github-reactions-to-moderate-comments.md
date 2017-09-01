---
layout:     post
title:      Using Github reactions to moderate comments
date:       2017-09-15 22:00:00
summary:    I turned Github reactions into a way to moderate which comments get posted to my blog. 
categories: codemonkey github jekyll javascript
commentIssueId: 29
---

A while ago I described a way to 
[turn Github issues into blog comment threads](). I've been using it
without issue, but it's fairly barebones; I never tried implementing
a text box and submit button via the blog post (users had to go to
the Github issue page) and it requires that you manually create 
issues for each post and specify YAML info to link an issue to a
particular post. There's a new tool called 
[utteranc.es](https://utteranc.es/) that looks pretty slick; it 
automates the issue linking and does fancy OAuth stuff to let
users submit comments from the blog post itself.

One thing utteranc.es doesn't do is allow you to moderate comments.
Granted, there isn't any mechanism to do this on the Github issues
themselves (that wouldn't make sense for their original purpose)
but it's easy to conceive of a way to, say, prevent certain comments
from making it onto your blog post (even if they remain in the Github
issue thread itself).

Github 
[reactions](https://github.com/blog/2119-add-reactions-to-pull-requests-issues-and-comments)
are a nice feature for collaborative work on code bases, but they have
limited utility for a blog. I figured that one could commandeer a 
particular reaction type to use as a way for moderating comments.
For this example, I decided to use a whitelist approach where 
comments would only get posted if I (Github username `mkoohafkan`)
gave the comment a `+1` reaction.

The snippet below scans through the comments of a given issue
and pulls only those that got a `+1` reaction from me. I made
heavy use of 
[this medium post](https://medium.com/@sungyeol.choi/making-multiple-ajax-calls-and-deciphering-when-apply-array-b35d1b4b1f50)
to handle multiple asynchronous AJAX requests. It might not be the 
most efficient code, but it works. Test it out by pasting the 
code chunk below into an HTML file and opening it in a browser.

```html
<html>
<head>
<script src="https://ajax.googleapis.com/ajax/libs/jquery/3.1.1/jquery.min.js"></script>
</head>

<body>

<div id="comments">
</div>


<script type="text/javascript">
  function loadComments(data) {
    var cuser = data.user.login;
    var cuserlink = data.user.html_url;
    var clink = data.html_url;
    var cbody = data.body_html;
    var cavatarlink = data.user.avatar_url;      
    var cdate = new Date(data.created_at);
    $("#comments").append(
       "<div class='comment'>" + 
          "<div class='commentheader'>" + 
            "<div class='commentgravatar'>" + 
              '<img src="' + cavatarlink + '" alt="" width="30" height="30">' + 
            "</div>" + 
            "<a class='commentuser' href=\""+ cuserlink + "\">" + 
              cuser + 
            "</a>" + 
            "<a class='commentdate' href=\"" + clink + "\">" + 
              cdate.toLocaleDateString("en") +  
            "</a>" +
          "</div>" + 
          "<div class='commentbody'>" + 
            cbody + 
          "</div>" + 
        "</div>"
    );
  }
  function getComments(data) {
    var results = [];
    var deferred = [];
    var deferreds = [];
    for (var i=0; i<data.length; i++) {
      deferred = $.ajax({
        url: data[i].reactions.url, 
        type: "get",
        data: {
          content: "+1"
        },
        headers: {Accept: "application/vnd.github.squirrel-girl-preview"},
        dataType: "json",
        success: function(result) {
          if(result.length < 1) {
            results.push(false)
          } else {
            for(var j=0; j<result.length; j++) {
              if (result[j].user.login == "mkoohafkan") {
                results.push(true);
              } else {
                results.push(false);
              }
            }
          }
        }
      });
      deferreds.push(deferred);
    }
    $.when.apply($, deferreds).then(function() {
      for(var k=0; k < results.length; k++) {
        if (results[k]) {
          loadComments(data[k]);
        }
      }
    })
  }
  
  $.ajax({
    // url should refer to blog post issue number
    url: "https://api.github.com/repos/mkoohafkan/mkoohafkan.github.io/issues/29/comments",
    type: "get",
    headers: {Accept: "application/vnd.github.squirrel-girl-preview.html+json"},
    dataType: "json",
    success: function(msg){
      getComments(msg);
   }
  });
</script>


</body>

</html>
```

You'll see that the first comment I left on this post gets loaded 
by the HTML file, but the second one does not; this is because I
only gave a `+1` to the second comment. You could easily modify the 
code to check for `-1` and reverse the true/false values of the 
`results.push` statements to implement a comment blacklisting 
approach. Neat, right? 

The main limitation right now is that the reactions API is still in
preview, and might change without warning; I don't want to push this
to my blog only to have my comment system suddenly break. But it's a
proof of concept, and should be straightforward to implement once the
reactions API is finalized.


