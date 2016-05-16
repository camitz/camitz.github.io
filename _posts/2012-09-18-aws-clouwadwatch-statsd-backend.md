---
date: 2012-09-18 14:02
title: Aggregating monitoring statistics for AWS Cloudwatch
Type: post
tags: aws, cloudwatch, amazon, node.js, log4net, cloudwatchappender, monitoring, awssum
---

I started out this blog connecting [log4net](http://logging.apache.org/log4net/) to [CloudWatch](http://aws.amazon.com/cloudwatch/), a monitoring service in Amazon's AWS cloud. The idea was to turn all those log4net log events that you might have in your code into graphs on CloudWatch, while modifying only the config.

The result was [CloudWatchAppender](https://github.com/camitz/CloudWatchAppender) which was my first ever project on GitHub (and NuGet) and which I've developed in a few crucial steps since. It has matured to a feature rich little lib and can do alot more than count events. It has seen a humble 66 downloads from NuGet. Totally worth the effort, if you ask me, not least since I use it in production extensively.

One caveat that was immediately realized that you had to be very selective with the events you send off. One, it'll quickly clog up your bandwidth. How does ten, twenty, a hundred or even one http-requests to CloudWatch per incoming page request sound? Secondly, CloudWatch imposes it's own limit as well.

This only diminished it's usefulness but we would like a way to aggregate the data and perhaps leverage CloudWatch support for sending statistics. This is something that you'd rather not do within a web application, typically with a lifetime in the order of a single request. The alternative is a more permanent service hanging around in the sidelines, acting as a middleman between your app and CloudWatch, sending statistics maybe a few times every minute.

Taking a look at whats out there, there is a fairly standard solution, or so it seems to me... for Unix environments. It's called [Statsd](https://github.com/etsy/statsd) + [Graphite](http://graphite.wikidot.com/). Statsd is an excellent and very lightweight aggregator written in <code>node.js</code>. 

Statsd comes ready for use with Graphite out of the box, which is an open source viewer for all kinds of statistics. It look's great from what I can tell. I wasn't able to install Graphite on my system.

As for Statsd, well it's node.js. Node.js = javascript so I was not intimidated, especially since node.js now fully supports Windows. The installer is downloadable off the official site.

This holds some promise if we can also find something node.js + AWS. Google turned up Awssum and in particular [node-awssum](https://github.com/appsattic/node-awssum) which is exactly what we're looking for.

So that's what we're going to do for this post: Connect log4net to Statsd to CloudWatch.

## How easy is this?

I think very. It's all install, configure and test save for a *backend* for Statsd. I've never done anything in node.js before although I regularly code in javascript and dojo.

First, if you haven't already, download and install <code>node.js</code>. Test it in a console window. Issue <code>node</code>. If that brings you into the direct interpreter then you're good to go.

Download and extract [Statsd](https://github.com/etsy/statsd) or clone it with [GitHub for Windows](http://windows.github.com/) if you're on a development system. Rename <code>exampleConfig.js</code> to <code>myConfig.js</code>.

In the console window navigate to where Statsd is and issue 
	
	node stats.js ./myConfig.js

You should now have 
	
	The server is up.

and some other bits of info.

Will it recieve data? Bring up that <code>config.js</code> again, replace the default backend with the one that outputs to console, set the flushinterval to 3 seconds. You've got something that looks like the following. I've removed some stuff that is of no consequence to us.

	{
	flushInterval: 3000
	, port: 8125
	, backends: [ "./backends/console" ]
	}
	
That will aggregate everything incoming on UPD on port 8125 and pass it on to the backend every 3 seconds. The backend is a pluggable piece of code. it can be a npm module, if you're familiar with that sort of lingo. In this case it's going to dump stuff to the console. Understandably the input has to be formated to suit Statsd. "gorets:1|c" seems an adequate choice, taken from the Statsd readme. That's a counter with increment 1 and it's called "gorets". Terminology for Statsd is bucket for the counter name.

## Test it

In Visual Studio we're going set up a new console project to test our gear. Create the project from a console template. Then put the following in <code>program.cs</code> and hit F5. It's basically just Microsoft example code.

	using System;
	using System.Net;
	using System.Net.Sockets;
	using System.Text;
	
	class Program
	{
	    static void Main(string[] args)
	    {
	        Socket s = new Socket(AddressFamily.InterNetwork, SocketType.Dgram,
	            ProtocolType.Udp);
	
	        IPAddress broadcast = IPAddress.Parse("127.0.0.1");
	
	        byte[] sendbuf = Encoding.ASCII.GetBytes("gorets:1|c");
	        IPEndPoint ep = new IPEndPoint(broadcast, 8125);
	
	        s.SendTo(sendbuf, ep);
	
	        Console.WriteLine("Message sent to the broadcast address");
	    }
	}

From Statsd you'll get something like

	17 Sep 13:40:08 - reading config file: ./myConfig.js
	17 Sep 13:40:08 - server is up
	Flushing stats at Mon Sep 17 2012 13:40:18 GMT+0200 (W. Europe Daylight Time)
	{ counter: { 'statsd.packets_received': 3, 'statsd.bad_lines_seen': 0 },
	  timers: { 'statsd.packet_process_time': '0', gorets: '3' },
	  gauges: {},
	  sets: {},
	  pctThreshold: [ 90 ] }
	
within 3 seconds.
	
I can't believe I contemplated coding up my on Windows service. This could not be any easier.

## Making a backend

How to roll a backend is outlined on the Statsd readme. You're listening to three events of which <code>flush</code> is relevant to us now. Starting with the config file let's start designing what we want. Add the following to config.js.

      , backends: [ "./backends/aws-cloudwatch-statsd-backend" ]
      , cloudwatch: {accessKeyId:'YOUR_ACCESS_KEY', secretAccessKey:'YOUR_SECRET_KEY', region:"EU_WEST_1"}

It's clear what we expect here. The first step is to copy <code>console.js</code>, i.e. the backend we've already tested, and rename it to <code>aws-cloudwatch-statsd-backend.js</code>. It will still work of course.

Using <code>console.js</code> we can just add the extra functionality while leaving the old code in place for tracing. Add lot's of debug outputs as well. To get <code>config.js</code> in, modify the constructor to look like

	this.config = config.cloudwatch || {};

in the appropriate place.

## Awesome

Node-awssum is an npm-module meaning it will install almost by itself. Issue <code>npm install -g awssum</code> and Node-awssum installs it to where everybody can find it. 

Browse the source code. In the example directory we're given a wealth of coding examples. For CloudWatch we find <code>list-metrics.sj</code>. Close enough for starters. We see that we need this

	var fmt = require('fmt');
	var awssum = require('awssum');
	var amazon = awssum.load('amazon/amazon');
	var CloudWatch = awssum.load('amazon/cloudwatch').CloudWatch;
	
at the top. The first of these includes is a console formatter used extensively by Awssum, it appears. You don't have to use it but it turns out to be very useful. Let's install it, too. <code>npm install -g fmt</code>.

The actual request should go in the <code>flush</code> event handler. The choice of variable names in <code>myConfig.js</code> at this point you'll realize was deliberately chosen to match with Awssum's. The passed object fits snuggly into the CloudWatch config with one exception that we'll get to in a moment.

    var cloudwatch = new CloudWatch(this.config);

    cloudwatch.ListMetrics(function(err, data) {
        fmt.msg("listing metrics - expecting success");
        fmt.dump(err, 'Error');
        fmt.dump(data, 'Data');
     });
	
Firing up now will produce and complaint about the region and that's because our region string is not what's expected. However, it does index into an array I found in the source code. The following line in the constructor will do the trick.

    config.cloudwatch.region = amazon[config.cloudwatch.region];

Now you'll have a backend that lists your metrics if you have them, plus what's left of the console backend, every time you get a <code>flush</code> event. It's not what we want but we're definitely getting somewhere. We've connected Statsd to CloudWatch which is more than half way. Before we get any further let's take a look at the log4net end of the chain.

## The UDPAppender

Someone meant for this to be done. log4net provides an appender targeting UPD services and it's all just configuration from here.

Halt the console app and hop into *NuGet manager* in Visual Studio and add the latest log4net library. <code>Program.cs</code> will look like this.
	
	using System.Threading;
	using log4net;
	using log4net.Config;
	
	class Program
	{
	    private static readonly ILog log = LogManager.GetLogger(typeof(Program));
	
	    static void Main(string[] args)
	    {
	        XmlConfigurator.Configure();
	
	        while (true)
	        {
	            log.Info("Counting. The message will be ignored.");
	 
	            Thread.Sleep(10);
	        }
	    }
	}
	
Notice, we've added some stress to the system. 100 calls a second.

The config may look like this:

	<?xml version="1.0" encoding="utf-8" ?>
	<configuration>
	  <configSections>
	    <section name="log4net" type="log4net.Config.Log4NetConfigurationSectionHandler, log4net"/>
	  </configSections>
	
	  <log4net>
	    <appender name="UdpAppender" type="log4net.Appender.UdpAppender">
	      <remoteAddress value="127.0.0.1" />
	      <remotePort value="8125" />
	      <layout type="log4net.Layout.PatternLayout" value="gorets:1|c" />
	    </appender>
	
	    <appender name="ConsoleAppender" type="log4net.Appender.ConsoleAppender">
	      <layout type="log4net.Layout.PatternLayout">
	        <conversionPattern value="%date [%thread] %-5level %logger [%ndc] - %message%newline"/>
	      </layout>
	    </appender>
	
	    <root>
	      <level value="ALL"/>
	      <appender-ref ref="ConsoleAppender"/>
	      <appender-ref ref="UdpAppender"/>
	    </root>
	  </log4net>
	</configuration>

We've add a console appender here just for the trace. Notice the layout element. We're not going to care about the actual event message. We're formatting everything to just "gorets:1|c", same as before. We're not even bothering to consider what metric name might be used, that'll be a feature for later.

Test it. You should have two console windows output lot's of text, one continuously, one every three seconds.

## PutMetricData

We'll skip ahead now. Unfortunately, there is no example code in Awssum for the AWS [PutMetricData](http://docs.amazonwebservices.com/AmazonCloudWatch/latest/APIReference/API_PutMetricData.html) call. But a file search turned up with header declaration with expected parameters and also some test code. Add some guessing, browsing some of the other examples and the online API reference for AWS and we may produces this.

	var metricDatum = {
	            MetricName : 'Gorets',
	            Unit : 'Count',
	            Value : metrics.counters.gorets
	        };
	
	cloudwatch.PutMetricData({
			MetricData : [metricDatum],
			Namespace  : 'Gorets'
		},
		function(err, data) {
			fmt.msg("Putting metrics");
			fmt.dump(err, 'Error');
			fmt.dump(data, 'Data');
		});
	};
	
We're not overreaching. This is no a finished product. The final result will put counter data on CloudWatch. The chain is finished and we did what we set out to do. The code below has added on line to the metricDatum object, for the sake of timestamp pickyness. Don't forget, AWS lives in UTC. 

	Timestamp: new Date(timestamp*1000).toISOString()

I've also removed the code from the original console backend we don't need.

	var util = require('util');
	
	var awssum = require('awssum');
	var amazon = awssum.load('amazon/amazon');
	var CloudWatch = awssum.load('amazon/cloudwatch').CloudWatch;
	var fmt = require('fmt');
	
	function CloudwatchBackend(startupTime, config, emitter){
	  var self = this;
	  
	  config.cloudwatch.region = config.cloudwatch.region ? amazon[config.cloudwatch.region] : null;
	  this.config = config.cloudwatch || {};
	
	  // attach
	  emitter.on('flush', function(timestamp, metrics) { self.flush(timestamp, metrics); });
	};
	
	CloudwatchBackend.prototype.flush = function(timestamp, metrics) {
	
	  var cloudwatch = new CloudWatch(this.config);
	
	var metricDatum = {
	            MetricName : 'Gorets',
	            Unit : 'Count',
	            Value : metrics.counters.gorets,
				Timestamp: new Date(timestamp*1000).toISOString()
	        };
	
	cloudwatch.PutMetricData({
			MetricData : [metricDatum],
			Namespace  : 'Gorets'
		},
		function(err, data) {
			fmt.msg("Putting metrics");
			fmt.dump(err, 'Error');
			fmt.dump(data, 'Data');
		});
	};
	
	exports.init = function(startupTime, config, events) {
	  var instance = new CloudwatchBackend(startupTime, config, events);
	  return true;
	};
	
Take a look at CloudWatch now, maybe something like this will show up.

![](https://dl.dropbox.com/u/1551997/statsd_backend.png)

I'll continue working this into something useful. I have already included it in production on cocoin.com. You can find it already on [GitHub](https://github.com/camitz/aws-cloudwatch-statsd-backend). By all means fork it and contribute or just suggest additions. The GitHub version has a different layout copied from another backend project for [MongoDB](https://github.com/dynmeth/mongo-statsd-backend.git). In the future it's meant to an npm module that you can install just as we did Awssum and Fmt.

A couple of branches I've been thinking is how we could integrate the system with CloudWatchAppender or even if this is desirable; and getting Graphite working on windows. I'd like to see a tutorial on that.

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
