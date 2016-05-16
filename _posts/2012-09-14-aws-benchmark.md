---
date: 2012-09-14 15:50
title: EC2 region benchmark
Type: post
tags: amazon, aws, ec2, benchmark, webpagetest
---

Ever thought about what region to use for what audience? Lot's of people must have but they keep their findings to themselves from what I can tell. 

Obviously if you want to target Japan, put you site in the Tokyo region. But if you want to target Vietnam or Thailand, like the client who put me up to crude benchmark did, then you may be at a loss.

I have a PhD but this is so unscientific. Interesting perhaps, nonetheless. 

I set up the same site on a number of different Regions in the AWS cloud. Then tested the first uncached page load using the very helpful [webpagetest.org](http://www.webpagetest.org/). Webpagetest.org let's you test your site as a browser client from a number of different sites accross the globe. Unfortunately not Vietnam and not Thailand. You can even script it if you want. 

The resulting graph looks like what you would get from hitting F12 in Chrome and opening the network tab.

Here are my results.

<table>
<tr><td colspan=2></td><td colspan=3>Access point</td></tr>
<tr><td colspan=2></td><td>Jiangsu</td><td>Tokyo</td><td>Amsterdam</td></tr>
<tr><td rowspan=5>Region</td><td>Ireland (eu-west)</td><td>55 s</td><td>8 s</td><td>4 s</td></tr>
<tr><td>Virginia (us-east)</td><td>10 s</td><td>6 s</td><td>5 s</td></tr>
<tr><td>Oregon (us-west)</td><td>8 s</td><td>5 s</td><td>6 s</td></tr>
<tr><td>Singapore (ap-southeast)</td><td>16 s</td><td>4 s</td><td>9 s</td></tr>
<tr><td>Tokyo (ap-northeast)</td><td>11 s</td><td>29 s</td><td>5 s</td></tr>
</table>

Disclaimer: Singapore is the original site and has some overhead, like SSL-encryption. I only did this once so that hardly qualifies as statistics. But it gives you an idea about what to expect when you launch a site intended for certain locality.

Someone should do an extensive benchmark. Maybe I should. Btw, take at [Google Page Speed](https://developers.google.com/speed/pagespeed/) if you haven't already.

<div id="disqus_thread"></div>
<script type="text/javascript">
/* * * CONFIGURATION VARIABLES: EDIT BEFORE PASTING INTO YOUR WEBPAGE * * */
var disqus_shortname = 'martincamitz'; // required: replace example with your forum shortname

/* * * DON'T EDIT BELOW THIS LINE * * */
(function() {
var dsq = document.createElement('script'); dsq.type = 'text/javascript'; dsq.async = true;
dsq.src = 'http://' + disqus_shortname + '.disqus.com/embed.js';
(document.getElementsByTagName('head')[0] || document.getElementsByTagName('body')[0]).appendChild(dsq);
})();
</script>
<noscript>Please enable JavaScript to view the <a href="http://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>
<a href="http://disqus.com" class="dsq-brlink">comments powered by <span class="logo-disqus">Disqus</span></a>
