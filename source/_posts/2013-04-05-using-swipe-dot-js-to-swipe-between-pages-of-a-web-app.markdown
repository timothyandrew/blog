---
layout: post
title: "Using Swipe.js to swipe between pages of a web app"
date: 2013-04-05 21:45
comments: true
categories: [howto, js, html, mobile, swipe.js]
---

I was working with [Steven](http://deobald.ca/) on [Jok](http://github.com/deobald/jok]) today.   

We were converting a two page (well 3, actually) page web application into a single page application that let you swipe between pages on mobile device but worked the same on a desktop browser.  

Here's how that's done.

First, download the [Swipe.js](http://swipejs.com/) plugin.

You need the markup for all your pages in a single HTML file, wrapped with two `divs` that Swipe uses.

```html
<div id='slider' class='swipe'>
  <div class='swipe-wrap'>
    <div class="page">
      Page 1
    </div>
    <div class="page">
      Page 2
    </div>
  </div>
</div>
```

Next, we need to setup Swipe.js.

```html
<script type="text/javascript">
    var mySwipe = Swipe(document.getElementById('slider'));
    mySwipe.setup();
</script>
```

We need some CSS styles too.

```css
.swipe {
  overflow: hidden;
  visibility: hidden;
  position: relative;
}
.swipe-wrap {
  overflow: hidden;
  position: relative;
}
.swipe-wrap > div {
  float:left;
  width:100%;
  min-height: 100%;
  position: relative;
}
html,body {
    height: 100%;
    margin: 0px;
    padding: 0px;
}
```

We've added styles to make each page have a `100%` height, so that our background-color spans the entire page.

This should allow page navigating between the pages by swiping on a mobile device.  
On a desktop, though, there's no way to switch between pages.

Swipe.js doesn't detect swipe-like events from the mouse, so we need to add buttons to the page for navigation.

```html
<div id="left-arrow" 
     style="font-size: 50px; position: absolute; left: 0%; top: 50%; z-index: 1000;"
     onclick="javascript:window.mySwipe.prev();">
&lt;
</div>
<div id="right-arrow" 
     style="font-size: 50px; position: absolute; right: 0%; top: 50%; z-index: 1000;"
     onclick="javascript:window.mySwipe.next();">
&gt;
</div>
```

This adds a button on each side of the page which allow navigating to the previous/next pages.

Pretty straight-forward so far. There's two problems with this approach, though.

- When a page is taller than the height of the viewport, a scrollbar shows up on all pages.  
  *This is not too bad. We could probably live with this.*
- If we scroll way down on a page, and then switch page, the new page will preserve the scroll position of the previous page. This can be particularly annoying if you have one really tall page and another short page.

We solved this problem by caching the scroll position for each page, and then scrolling to that position when the page was changed.


To hook into the page change event, we need to pass a function into the Swipe.js initializer.

```javascript
    var body = document.getElementsByTagName("body")[0];
    var otherScroll = 0;
    
    var cb = function(index, element) {
      oldScroll = otherScroll;
      otherScroll = body.scrollTop;
      body.scrollTop = oldScroll;
    };
    
    var mySwipe = Swipe(document.getElementById('slider'), { callback: cb });
    mySwipe.setup();
```

This works okay for a two page app. If you've got more pages, you'll have to cache the scroll positions for each page.

```javascript
    var body = document.getElementsByTagName("body")[0];
    var positions = [];
    var currentPage = 0;
    
    var cb = function(index, element) {
      positions[currentPage] = body.scrollTop;
      body.scrollTop = positions[index] || 0;
      currentPage = index;
    };
    
    var mySwipe = Swipe(document.getElementById('slider'), { callback: cb });
    mySwipe.setup();
```