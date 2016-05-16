---
date: 2012-10-30 11:02
title: Updated Cloudwatch backend for StatsD
Type: post
tags: aws, cloudwatch, amazon, node.js, log4net, cloudwatchappender, monitoring, awssum
---

A month ago I successfully chained together [log4net](http://logging.apache.org/log4net/), [StatsD](https://github.com/etsy/statsd) and [CloudWatch](http://aws.amazon.com/cloudwatch/). I put together a backend for StatsD in node.js called [aws-cloudwatch-statsd-backend](https://github.com/camitz/aws-cloudwatch-statsd-backend).

Yesterday I finally updated it into something I can actually use. I'm using it in production on [Cocoin](https://www.cocoin.com).

StatsD's capability of pushing metadata along with the request is limited. But you can parse the bucket name for information. I wanted to extract namespace and metric name. 

Currently I'm sending UDP packages of this style to count the requests on the site.

    App/Controller/Action/Request:1|c

This get's translated to a cloudwatch request with *App/Controller/Action* as a namespace and Request as the metric name.

Using the bucket name for metadata tampers with generality but doesn't blow it completely. You can still add on arbitrary backends.

To install it, use npm.

    npm install aws-cloudwatch-statsd-backend

Here is an example configuration.

	{
	    backends: [ "aws-cloudwatch-statsd-backend" ],
	    cloudwatch: 
	    {
	        accessKeyId: 'YOUR_ACCESS_KEY_ID', 
	        secretAccessKey: 'YOUR_SECRET_ACCESS_KEY', 
	        region: 'YOUR_REGION',
	        processKeyForNames:true
	    }
	}

You can also override both the *namespace* and metric name (*metricName*) in the config.

Full documentation on the [npm](https://npmjs.org/package/aws-cloudwatch-statsd-backend) and [GitHub](https://github.com/camitz/aws-cloudwatch-statsd-backend) pages.

My other monitoring project, [CloudWatchAppender](https://github.com/camitz/CloudWatchAppender), and appender for log4net wihout aggregation, has now been downloaded 236 times. Very pleased with that. 

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
