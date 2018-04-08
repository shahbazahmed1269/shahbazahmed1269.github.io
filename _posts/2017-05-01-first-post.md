---
layout: post
title: First Post
comments: false
---

Hi! Welcome to my blog. I created this blog to motivate myself to program regularly and share some of my experiences along the way.

This is my fist post. I wanted to create [this blog](http://shahbazahmed.me) as a motivation to learn programming and share my experiences. Since my school days I have a habit of learning things better if I make notes of it. And hence blogging about whatever I learn would help me learn better and at the same time allow me to share my learnings. My interest lies in the following topics:
`backend development`, `golang`, `android development` and `raspberry pi`. In the future posts I would be sharing my learnings and experiences in various topics depending on whatever I am exploring at that moment. :smile:

{% if page.comments %} 
   <div id="disqus_thread"></div>
    <script>
    var disqus_config = function () {
    this.page.url = PAGE_URL;  // Replace PAGE_URL with your page's canonical URL variable
    this.page.identifier = PAGE_IDENTIFIER; // Replace PAGE_IDENTIFIER with your page's unique identifier variable
    };
    (function() { // DON'T EDIT BELOW THIS LINE
    var d = document, s = d.createElement('script');
    s.src = 'https://shahbazahmed.disqus.com/embed.js';
    s.setAttribute('data-timestamp', +new Date());
    (d.head || d.body).appendChild(s);
    })();
    </script>
    <noscript>Please enable JavaScript to view the <a href="https://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>                        
{% endif %}