---
layout:     post
title:      Using self-signed certificates with Git
date:       2018-09-12 07:45:00
summary:    My new organization's self-signed certificates weren't playing nice with Git. Here's how I fixed the problem.
categories: infomaniac git ssl certificates
commentIssueId: 35
---

So I started a new job recently (woo!) and have been going through my 
rites of passage, namely cleaning the dust bunnies out of my cubicle,
raiding the supply closet for pens and notepads I'll probably never use, 
and installing my favorite software on my workstation. I've developed a lot of tools
over the years, and I use a number of unpublished or development versions
of R packages as part of my regular workflow. Getting Git set up typically 
happens right after installing my favorite IDE (which these days is 
[Visual Studio](https://visualstudio.microsoft.com/vs/) and its 
[Data Science workload](https://docs.microsoft.com/en-us/visualstudio/rtvs/data-science-and-analytical-applications-workload)).
Barring any complications with getting administrator rights on my work 
computer, setting up Visual Studio and Git is straightforward.

Imagine my surprise, then, when I tried to clone one of my Github 
repositories and instead got the following error:

>fatal: unable to access '[repo name]': SSL certificate problem: self 
signed certificate in certificate chain

What the heck?

Okay, so the message is at the same time clear and not-clear: my 
organization uses self-signed certificates (pretty normal) and it's
interfering with Git (pretty weird). So what do I do about it?

It took a few rounds of googling, but I eventually figured out that I
had to manually add my organization's self-signed certificates 
to the certificate library used by Git. Doing this required three things: 
*(1)* access to the Git command prompt, e.g. by installing 
[Git for Windows](https://gitforwindows.org/),
*(2)* Access to the Microsoft Management Console, and 
*(3)* a text editor.
I cobbled together the procedure below from two unrelated help pages
([one](https://specopssoft.com/support-docs/specops-password-reset/reference-material/installing-the-self-signed-ssl-certificate-into-the-trusted-root-certificate-authorities-store/), 
[two](https://writeabout.net/2017/02/03/git-for-windows-with-tfs-and-ssl-behind-a-proxy/)).

The first thing you need to do is figure out what certificates are 
actually causing the problem. You can get a good idea by going to 
the Git prompt and typing

```
openssl s_client  -connect www.github.com:443
```

This will output a bunch of crud, but you'll get a non-zero return 
code (I got 19) and buried in the output will be references to the 
offending self-signed certificates. The next step is to actually *find* 
the certificates on your machine. You can do this with the Microsoft 
Management Console (type `mmc` in the command prompt). 
Open the Microsoft Management Console and add the "Certificates" snap-in
via File-->Add/Remove Snap-ins, then select "Certificates" from the 
list of available snap-ins and specify "Computer account". This adds a 
directory in the tree on the left-hand side of the console. From here 
it's a game of hide-and-seek; I found the certificate I was looking for 
in Trusted Root Certification authorities/Certificates but your might live
somewhere else.

Once you find your certificate, you need to export it. Specifically, you 
need to export it in format **Base-64 encoded X.509 (.CER)**. Save the 
file wherever you want, we'll be done with it in a minute.

Open up the exported certificate with notepad (or even better, notepad++)
and you'll see something like

```
-----BEGIN CERTIFICATE-----
...a bunch of garbage...
-----END CERTIFICATE-----
```

This is the certificate in all its textual glory. The final step is to 
manually add this certificate to your list of trusted certificates.
First, find your certificate library. It's probably somewhere like 
`C:\Program Files\Git\usr\ssl\certs\ca-bundle.crt`, but you can find it 
by going to the Git prompt and entering the command

```
git config --global http.sslCAInfo
```

which will return the path of the certificate library Git uses. Find 
this file and open it up in notepad; You'll see a number of blocks like 
the one above. Simply paste the contents of the self-signed certificate 
into this file and save it.

Once I added the self-signed certificate to the certificates 
library, cloning and pushing to repositories started working 
normally. Crisis averted! Now back to the dust bunnies...

