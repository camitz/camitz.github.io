---
date: 2012-08-09 11:03
title: Improving the CloudWatch Appender
Type: post
permalink: improving-cloudwatch-appender
tags: amazon, AWS, cloudwatch, log4net, appender
---

*This is a follow-up to [this](http://blog.simpletask.se/post/awscloudwatch-log4net-appender) post, describing the original idea. This post seeded a project based on [Github](https://github.com/camitz/CloudWatchAppender). Go ahead and contribute.*

# Some much needed improvements

In the last post I started working on something I'd always thought needed but didn't find, a log4net appender targeting AWS CloudWatch, so that all the log messages scattered around my code base could be directed to a nice graph in my CloudWatch console.

What I quickly realized was that, hey, those requests are taking a while. I you wait for them, that's going to cost you a half second. If this is a website you're not going to want to use it for anything other than extraordinary events, like errors. Fine. Even so we can still do with a little asynchronicity.

Also we discussed getting some more information into the request, primarily, values other than 1 and units other than count. Perhaps statistics like sum, min and max. It would be nice if we could specify this in the log event if we wanted and also in the config, of course. Still don't want to change the log messages unless we have to.

We'd also like to set the name and namespace and there is also something known as dimensions, like instance id. Dimensions are of limited use, in my opinion, until Amazon allows custom metrics to be aggregated by dimensions. Anyhow, we'd like to specify those somehow.

Next, say we do want to log something very often, like page loads. Then we need an aggregator, a background service that receives our data points and aggregates it to a request to CloudWatch at most once a minute.

That last one seems like a dauntingly big project, doesn't it and something somebody must have thought about already? I did do some research and it seems like the standard solution is statsd + Graphite. That would be Unix. The recommended solution for Windows would seem to be to startup a Unix VM on your server with the moral, don't reinvent the wheel. That seems a bit of an overshoot to me, particularly since I'd still have to code a sink replacing Graphite with CloudWatch. Now, I do have some experience with MSMQ so we'll see if I can't work something out sometime in the future. What do you know, there happens to be both an MSMQ appender and a UDP appender for me to use.

This post won't a complete tutorial. I'll just skim through most of the details. You can download the result at the bottom.

# Async

Let's start with asynchronicity. I'm still using .Net 4.0, which you'll notice by my old fashioned asyncs and wait. But it works just fine.

The request to CloudWatch is made in a separate task with Task.Factory.StartNew. That's all you need. The only reason I added the structure for keeping track of my tasks was so that my new example project, ContinuousTicks, could wait out the requests before exiting. For doing that we need a thread safe collection, obviously. I'm using ConcurrentDictionary with task IDs as keys.

I tried this with 10000 data points. As you can see below, everything was logged in about 3 s and that included console output. The requests were completed in the 1 minute following. 

![asd](https://dl.dropbox.com/u/1551997/cloudwatchappender_screenshot3.jpg)

Now I haven't tried this on a web server, but it will perform worse, perhaps by an order. But this gives you an idea just how many log events to CloudWatch you can send. You need to be careful with your thread pool. Extraordinary errors are certainly fine and temporarily debug messages, just to sample something for a short while.

What's more, there's a limit over at amazon. I did at one point get an exception saying that my rate had been exceeded. You need to worry about that to. I'm not handling that error well at the moment, just outputting the message to the console and carrying on. The request is dropped.

# Values, units and stats

So far, the unit has been Count and the value 1. Perhaps you'd like to plot the sizes of photos being uploaded to your site. Well, I'm going set the appender to look for tuples like this: 15.4 Megabytes. That is a number followed by one of the [units](http://docs.amazonwebservices.com/AmazonCloudWatch/latest/APIReference/API_MetricDatum.html) recognized by CloudWatch. I'm also looking for stuff like this Value: 15.4 Megabytes, Value: 14.50 Unit:  Megabytes or just Value: 15.4. Also SampleCount:150, Sum: 200 Gigabytes, Minimum: 10, Maximum: 50 for specifying statistics.

These should all be settable in the config which will override anything present in log message. The rational, as always, this should work on existing. You shouldn't have to update all you event logs to exploit this library.

If this seems like a hassle you can also pass an object with this information. It will act just like the MetricDatum does, added the Message member. Log4net lets you plug in an object renderer for any other appenders wishing to consume these log events. This is getting very close to a reversal of the original idea, which was, "Wouldn't it be nice for our log events to be posted to CloudWatch." We're approaching, "Let's direct some of our CloudWatch metrics to log4net." Three cheers for flexibility.

Why not Average? CloudWatch will try and calculate this for you, based on single point values and Sum/SampleCount. You switch to pre-aggregated statistics sets when it's impractical to report singe point data. For example we discussed page loads in the original post. Read up on [this](http://docs.amazonwebservices.com/AmazonCloudWatch/latest/DeveloperGuide/cloudwatch_concepts.html) for more info an statistics.

# Config

So let's get to the config part first. Log4net allows you to pass arbitrary parameters to the appender in the config. These will act as defaults until they are overridden. log4net will try to set the corresponding properties in our appender.

    <param name="Unit" value="Megabytes"/>
    <param name="Value" value="0.01"/>

The following is identical.

    <unit value="Megabytes"/>
    <value value="0.01"/>

Are datum now looks like this.

    var data = new List<MetricDatum>
       {
           new MetricDatum()
               .WithMetricName(name)
               .WithUnit(unit)
               .WithValue(value)
       };

Notice that we've also got a name variable and in fact I added nameSpace as well, which goes in the request as it is supposed to be common for all the data points add to it. In a similar fashion you can set name and namespace with parameters to the appender.

**Warning** If you download my library, you'll have the option of setting, UseLoggerName. The name and namespace, unless overridden, will be computed from the logger name, typically the long name of the type that emitted it. CloudWatch will create a new metric for every namespace+name provided and this could put you back dollars if you're to generous. Set the parameters.

In my finished project I've also added support for statistics and dimensions. For statistics use variable names sum, minimum, maximum and samplecount as specified [here](http://docs.amazonwebservices.com/AmazonCloudWatch/latest/DeveloperGuide/cloudwatch_concepts.html) in config, in the event message or CloudWatchLogObject. Specify dimensions like this.

    <dimension0 type="Amazon.CloudWatch.Model.Dimension">
        <name value="BetaWebSite"/>
        <value value="2" />
    </dimension0>

Those are enumerable up to 9, ten in total.

**Warning** Again, CloudWatch creates a custom metric for every combo of dimensions you supply. Be careful so you don't increase Amazon's revenues by a measurable amount.

Would we ever want to put the EC2 instance id in a dimension? Most certainly. If instead of BetaWebSite you put InstanceID and leave the value empty, then I will supply you with it. This is done with an http request to Amazon, retrieving [instance metadata](http://docs.amazonwebservices.com/AWSEC2/latest/UserGuide/AESDG-chapter-instancedata.html). The following will do this.

    return new StreamReader(
        WebRequest.Create("http://169.254.169.254/latest/meta-data/instance-id")
            .GetResponse()
            .GetResponseStream(), true)
        .ReadToEnd();

You can error check if you like. You'll get a WebException if you're not actually doing this from within an EC2 instance.

# The parser

The parser uses a regular expression to pick up triples like this. 

    log.Info("A user upload a new photo of size Value: 2.5 Kilobytes");

It's not very smart at the moment so "Size of photo: Value: 2.5 Kilobytes" will not work as expected. It's not a real syntax parser, but maybe it will be one day.

But it *is* kind of smart. It'll read the string from left to right and try it's utmost to get data out, rather than fail. 

As soon as the parser can't fit new info in what's it's been doing so far, it just starts a new data point. So the following

    log.Info("A user upload a new photo of size Value: 2.5 Kilobytes, Unit: Megabytes");

Will create two data points, the second with the default value 1.0. CloudWatch will interpret that fine. Will you though? These days the spirit is, I find, to be permissive rather than strict. Computers are strict. Programmers need to step up and try to find out what users want. At any rate I decided that I should not try to put in more restrictions that CloudWatch does. Anything the CloudWatch accepts and graphs, we can parse.

I'm continually developing the project as open source on [Github](https://github.com/camitz/CloudWatchAppender). Hence, there is no just for this blog post. It's all in the project. Feel free to contribute to the project.



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
