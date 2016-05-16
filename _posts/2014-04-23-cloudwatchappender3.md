---
date: 2014-04-23 16:00
title: Introducing the Buffering and Aggregeting CloudWatch Appender
Type: post
permalink: buffering-aggregating-cloudwatch-appender
tags: amazon, AWS, cloudwatch, log4net, appender, bufferingappender
---

The CloudWatch Appender for log4net is an appender that targets AWS CloudWatch. So if you have any kind of service on the Amazon cloud you can easily forward your log messages via the AWS Api to show on pretty graphs in your AWS consoles. By default the points in the graphs represent a hit count but with purposefull configuration in config files and in patterns of the messages themselves, we can show values and statistics in any unit supported by AWS.

One of the limitations of the CloudWatch Appender has been that you definitely want to do some configuration and filtering. If you redirect *all* your log messages to Cloudwatch then you would very quickly choke your service waiting for the API. In fact AWS limits your calls, but you'd still be making the requests trying.

An update some time ago made a slight improvement in that the requests were collected and performed asynchronously, but that only meant that the appender was able to handle bursts better. A stream was of continuous events were solved with the introduction of an event rate limiter - by dumping them into the bit bucket.

Why it took me so long to implement the promised buffering appender, I don't know. It works very well and was not hard to do.

Instead of sending a single API request per log message, the BufferingAggregatingCloudWatchAppender collects event messages in the same way as the BufferingForwardingAppender that comes with the package does. When a condition is met, the level of the message or a maximum buffer size, the collected events are processed and aggregated data according to NameSpace, MetricName and Dimensions, then sent to CloudWatch as a statistic set.

The overhead is minimal.

# Don't filter

This means that installation has never been simpler.

    <log4net>
       <appender name="CloudWatchAppender" type="CloudWatchAppender.BufferingAggregatingCloudWatchAppender, CloudWatchAppender">
            <bufferSize value="1000"/>
            <accessKey value="YourAWSAccessKey" />
            <secret value="YourAWSSecret" />
            <endPoint value="eu-west-1" />
        </appender>

        <root>
          <appender-ref ref="CloudWatchAppender" />
        </root>
    </log4net>

Say a single request to your web application on average generates 50 log messages and you have 100 requests a minute... well, then using the regular CloudWatchAppender you wouldn't have a web application anymore, just 500 http requests a minute.

At a price of 1 cent per 1000 API request, Amazon seems pretty generous, more of a don't get carried away limit. If you're paying for it, you're doing it wrong. And there's a request limit on top of that. But if there was no limit, the above example would amount to 216,000,000 requests per month. 

Uhm... $2159, ok? But, hey, first mill' free!

Strong case for BufferingAggregatingCloudWatchAppender. Using BufferingAggregatingCloudWatchAppender, you'll be making about one request every two minutes. Async. That's nothing.

So, no rush. Just put the above in your config, it'll work, it won't cost you. It will be somewhat usefull. When you get round to it, you can start configuring NameSpaces, MetricNames, Dimensions and working with units, pushing info that somebody needs to know. You will incur a cost in dollars eventually but probably not in terms of bandwidth and waiting for http requests.

So use it and don't look back.

# Evaluators

log4net supports 3 evaluators out of the box for flushing the buffer. The following example uses TimeEvaluator.

      <lossy value="true" />
      <evaluator type="log4net.Core.TimeEvaluator">
        <interval value="60"/>
      </evaluator>

Regardless of the saturation level of the buffer, the events will be processed and sent to CloudWatch once a minute. Using lossy mode, the buffer won't be processed when filled. The oldest events are dropped until the evalutator is triggered.

I can think of several custom evaluators specifically for use with CloudWatch, but that's for later.

# Lot's of improvements

I've made some other improvements aswell. Some around the asynchronous requests - although it's probably not perfectly comme-il-faut and it's still very .NET 4.0 style. Some general refactorings have been made, mostly beneficial for me.

Debug outputs and error messages from CloudWatchAppender were previously passed to the TraceListener system. Now they are routed to log4net's internal scheme.

One of the things already in place is acutally a bug fix. It was a breaking change and released as such but I decided to do it because I couldn't fathom anyone using the feature in the wasy provided. It has to do with the augmented logger function and negative precision specifier. Read more [here](https://github.com/camitz/CloudWatchAppender/blob/master/README.md#augmented-logger-functionality).

Do stay tuned for upcoming releases. Here are some more things I have planned for the very near future.

## Unit converter

Microseconds, milliseconds. Same, but different. Kilobytes, megabytes, even Gigabits, same thing there. Amazon supports these units but doesn't seem to know how to convert them. If you've made but a single request for a particular MetricName for a certain NameSpace, then you unit is set in stone for the next two weeks. It doesn't matter what you send, AWS will interpret thos values as representing the original unit. That means you need to keep track of your units and not log Kilobytes, Bytes and Megabytes for the same metric. 

It's not published yet, but soon, CloudWatchAppender will help you out. 

Firstly it will convert between different prefix versions of the same unit, for example Kilobytes end Megabytes. The buffering appender, if it's a freshly created appender, will find the unit with lowest prefix and convert all other values to that it.

Secondly, between requests, it'll keep track of that unit for as long as it's alive. So make one call with Kilobytes and later on one with Megebytes, you're going to get Kilobytes in the plot.

If the web app is recycled, or it's separate thread, or server then it'll this feature is not 100 % correct. In most cases it'll work out fine though since usually calls are made pretty much the same way and order and surprised only happen unless you make changes in your code.

But just to be on the safe side, if it doesn't know, CloudWatchAppender will make a get request to the API and make sure. It will find our what unit is used for a certain metric and convert everything to that.

This last feature is further down the line but it will come eventually. And of course you can turn it off.

## Metric data per request limit

When using the unbuffered appender, we were burdening our requests with 1-2 metric data, as the data carrying objects are known. With the buffering appender we make one request per NameSpace and add to those as any metric data as we need, usually one per combination of MetricName and Dimensions used. I wasn't aware there was a limit imposed on how may MetricDatum objects can be added to a request, but there is. It's 20. Sooner or later someone will hit it, so that needs to be taken care of.

## Auto config

Sample config will be added to the nuget package once I learn how.

## Support for StandardUnit

Say you could do something like this?

      <standardunit type="Amazon.CloudWatch.StandardUnit">
           <value value="Kilobytes"/>
      </standardunit>

 Would you prefer it to the current?

     <unit value="Kilobytes" />

Maybe. Maybe not. It's just that I introduced the SDK's StandardUnit in every other place throughout the code to exploit its built in validation of the string value, instead of maintaining my own list. So why not use StandardUnit in config as well? Seems like alot more typing, but the main issue is another. Log4net requires StandardUnit to have a parameterless constructor, which it doesn't. 

So to support this feature of questionable value you'd need to modify the source code and put in for a pull request with the people behind log4net.

...

You bet I did!!




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
