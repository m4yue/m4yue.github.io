---
layout: default
title: Chrome MIME Sniffing
---

## Chrome MIME Sniffing

> chrome 浏览器判断如何渲染页面内容，来自chromium源代码注释。
> https://cs.chromium.org/chromium/src/net/base/mime_sniffer.cc?sq=package:chromium&dr=CS&l=5


Detecting mime types is a tricky business because we need to balance
compatibility concerns with security issues.  Here is a survey of how other
browsers behave and then a description of how we intend to behave.

### HTML payload, no Content-Type header:
* IE 7: Render as HTML
* Firefox 2: Render as HTML
* Safari 3: Render as HTML
* Opera 9: Render as HTML

Here the choice seems clear:

`Chrome: Render as HTML`

### HTML payload, Content-Type: "text/plain":
* IE 7: Render as HTML
* Firefox 2: Render as text
* Safari 3: Render as text (Note: Safari will Render as HTML if the URL
                                  has an HTML extension)
* Opera 9: Render as text

Here we choose to follow the majority (and break some compatibility with IE).
Many folks dislike IE's behavior here.

`Chrome: Render as text`

We generalize this as follows.  If the Content-Type header is text/plain
we won't detect dangerous mime types (those that can execute script).

### HTML payload, Content-Type: "application/octet-stream":
* IE 7: Render as HTML
* Firefox 2: Download as application/octet-stream
* Safari 3: Render as HTML
* Opera 9: Render as HTML

We follow Firefox.

`Chrome: Download as application/octet-stream`

One factor in this decision is that IIS 4 and 5 will send
application/octet-stream for .xhtml files (because they don't recognize
the extension).  We did some experiments and it looks like this doesn't occur
very often on the web.  We choose the more secure option.

### GIF payload, no Content-Type header:
* IE 7: Render as GIF
* Firefox 2: Render as GIF
* Safari 3: Download as Unknown (Note: Safari will Render as GIF if the
                                       URL has an GIF extension)
* Opera 9: Render as GIF

The choice is clear.

`Chrome: Render as GIF`

Once we decide to render HTML without a Content-Type header, there isn't much
reason not to render GIFs.

### GIF payload, Content-Type: "text/plain":
* IE 7: Render as GIF
* Firefox 2: Download as application/octet-stream (Note: Firefox will
                             Download as GIF if the URL has an GIF extension)
* Safari 3: Download as Unknown (Note: Safari will Render as GIF if the
                                       URL has an GIF extension)
* Opera 9: Render as GIF

Displaying as text/plain makes little sense as the content will look like 
gibberish.  Here, we could change our minds and download.

`Chrome: Render as GIF`

### GIF payload, Content-Type: "application/octet-stream":
* IE 7: Render as GIF
* Firefox 2: Download as application/octet-stream (Note: Firefox will
                             Download as GIF if the URL has an GIF extension)
* Safari 3: Download as Unknown (Note: Safari will Render as GIF if the
                                       URL has an GIF extension)
* Opera 9: Render as GIF

We used to render as GIF here, but the problem is that some sites want to
trigger downloads by sending application/octet-stream (even though they
should be sending Content-Disposition: attachment).  Although it is safe
to render as GIF from a security perspective, we actually get better
compatibility if we don't sniff from application/octet stream at all.

`Chrome: Download as application/octet-stream`

Note that our definition of HTML payload is much stricter than IE's
definition and roughly the same as Firefox's definition.
