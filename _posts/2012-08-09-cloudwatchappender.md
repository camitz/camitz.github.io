---
date: 2012-08-09 11:03
title: A CloudWatch Appender for log4net
Type: post
permalink: AWSCloudWatch-log4net-appender
tags: amazon, AWS, cloudwatch, log4net, appender
---

*This post seeded a project based on [Github](https://github.com/camitz/CloudWatchAppender). Go ahead and contribute. It's also on [NuGet]("https://nuget.org/packages/CloudWatchAppender).*

A log4net appender for Amazon AWS CloudWatch, why?

Well, you're logging, right? If you own a site, you should be logging pretty much everything. You'll use it for debugging, for optimizing and not least for security - detecting and finding the source of unusual behavior.

And I assume you're also using some kind of monitoring, a dashboard with cpu load, network i/o and new user applications. Right? How else would you detect the unusual behavior mentioned above in a manner timely enough to mitigate the damage?

I use [log4net](http://logging.apache.org/log4net/ "log4net") for logging, a system which aggregates log messages from anywhere in the code, in a hierarchical manner, and outputs data to text files, databases, event logs... There are lots of appenders for all kinds of sinks. And you can write your own! Really simple too. So me, I just code away, and when something seems important enough to let someone know about it, I put a log message in there. It's a low threshold. The message will end up in the root log file and if later on I decide I don't want it there, I can just edit the web.config and divert it to somewhere else, or turn it off.

I use [CloudWatch](http://aws.amazon.com/CloudWatch/ "CloudWatch") for monitoring. Makes sense, since my site is on Amazon EC2. I don't have to. There are other services, like [Geckoboard](http://geckoboard.com/ "Geckoboard"). I get a dashboard with graphs, giving me realtime info about what's going on. I can set up alarms for unusual behavior. I also use alarms to make sure my services are up. That means I send CloudWatch a tick once every minute and CloudWatch will text me if it doesn't get them. Contrary to what you might think, it makes you sleep better.

I think I've answered my question. I've riddled my code with log outputs and some of them make sense for me to publish to my dashboard, not as messages, of course, but as statistical observations. Hi, we passed this line of code once, it's 11.45 am, beep. You can pass numerical data, too, but I'll save that summary for another post.

# Getting started

Let's just start by setting up a framework for proof of concept, make sure what works and what doesn't.

Create a new console project in Visual Studio. I called mine RandomTicks. 

Add a new item to your project, an application configuration file.


You should be using NuGet Package Manager. In it, search for log4net and add it to your project. The following may or may not get added automatically to your app.config. If it's not there, add it.

    <configSections>
        <section name="log4net" type="log4net.Config.Log4NetConfigurationSectionHandler, log4net" />
    </configSections>

This is not a post about setting up and using log4net, other blogs do that well. 

This is what my console app looks like. 

	using System;
	using System.Threading;
	using log4net;
	using log4net.Config;

	namespace RandomTicks
	{
	    internal class RandomTicks
	    {
            private static readonly ILog log = LogManager.GetLogger(typeof(RandomTicks));
            
            private static void Main(string[] args)
            {
                XmlConfigurator.Configure();

                var random = new Random();

                while (true)
                {
                    if (random.NextDouble() < .3)
                        log.Info("A tick!");

                    Thread.Sleep(100);
                }
            }
	    }
	}


So, every 10th of a second, we're deciding with a 30 % probability whether or not to log something, and then do it. I then configured, so I know we're sending of the log messages.

	<log4net>
	    <appender name="ConsoleAppender" type="log4net.Appender.ConsoleAppender" >
	      <layout type="log4net.Layout.PatternLayout">
		        <conversionPattern value="%date [%thread] %-5level %logger [%ndc] - %message%newline" />
	      </layout>
	    </appender>

	    <root>
	      <level value="INFO" />
	    </root>
	</log4net>

Hit F5 and you should be getting messages to your console, like this, with a frequency of about 200 times a minute, on average.

![](https://dl.dropbox.com/u/1551997/cloudwatchappender_screenshot2.jpg)


#The appender

Writing your own appender is real easy, as stated by [Robert Prouse](http://www.alteridem.net/2008/01/10/writing-an-appender-for-log4net/ "Robert Prouse's blog"). I mostly followed his example. I still got it wrong at first but that's because I get ahead of myself and don't follow instructions. Follow instructions.

Add a new class library to your solution. Reference it from your test app, just to make sure it gets copied to your bin directory when you compile.

With Package Manager, add references to log4net and also the AWSSDK library. You may also want a reference to System.Configuration, just so you can put your AWS credentials in the config-file.

AppenderSkeleton, provided by the log4net library, makes it a sinch. Simply subclass it and override the Append method. Something like this.

	namespace Log4NetCloudWatchAppender
	{
	    public class Log4NetCloudWatchAppender : AppenderSkeleton
	    {
            protected override void Append(LoggingEvent loggingEvent)
            {
              
            }
	    }
	}

Let's put a break point in there and see if we can get it to hit. First we need to add the appender to the configuration. 

    <appender name="CloudWatchAppender" type="Log4NetCloudwatchAppender.Log4NetCloudwatchAppender, Log4NetCloudwatchAppender">
    </appender>

Add it to the root logger with 

    <root>
      <level value="INFO" />
      <appender-ref ref="ConsoleAppender" />
      <appender-ref ref="CloudWatchAppender" />
    </root>

Hit F5 and check that your breakpoint is being hit.

# The AWS client

CloudWatch works by way of a restful web API, same as everything on AWS. For .NET you have the AWSSDK that makes everything, well not easy, but it removes some layers of complications while adding its own. Getting the AWS client to work could be simple, but I think for most the first time, it's a headache. You need your credentials, you need a service endpoint, etc. 

My story is that I added a timestamp to the request. I spent the next hour trying to figuring our why I wasn't seeing any data, then the hour after that, trying to figure out why I was seeing data, even though my app was disabled. One of those times when debugging didn't help. Taking a break, stepping outside, did. CloudWatch is set to UTC and refuses to display anything that it thinks hasn't happened yet. Use UCT or just don't bother sending a timestamp at all. The client will put one in there all on its own. The lessons learnt is for another post, maybe.

Anyway, get your endpoint from [here](http://docs.amazonwebservices.com/general/latest/gr/rande.html#cw_region). Log in to AWS and get your credentials from [here](https://portal.aws.amazon.com/gp/aws/securityCredentials). That's your access key id and your secret access key. Preferably you won't use your main credentials but rather set up user with the [IAM](http://aws.amazon.com/iam/) feature and give it access just to post CloudWatch requests.

Put this in your Append-method and add the appropriate includes.

    var CWClient = AWSClientFactory.CreateAmazonCloudWatchClient(
          ConfigurationManager.AppSettings["AWSAccessKey"],
          ConfigurationManager.AppSettings["AWSSecretKey"],
          new AmazonCloudWatchConfig { ServiceURL = "https://monitoring.eu-west-1.amazonaws.com" }
        );

    var data = new List<MetricDatum>
                   {
                       new MetricDatum()
                           .WithMetricName("RandomTicks")
                           .WithUnit("Count")
                           .WithValue(13)
                   };

    try
    {

        var response = CWClient.PutMetricData(new PutMetricDataRequest()
           .WithNamespace("RandomTicks")
           .WithMetricData(data));

    }
    catch (Exception e)
    {
        //Don't log this exception :)
    }

Hit F5. Log in to AWS and go to CloudWatch. After a moment or two data will be available under a new "metric" called RandomTicks. You should be seeing something like this.

![asd](https://dl.dropbox.com/u/1551997/cloudwatchappender_screenshot.jpg)

# AWS stats 

To note from the code above is that we've chosen, from a fixed set of units, the dimensionless "Count". In another post I will explore different units, but for now, we're just posting single observations. I'm using the value 13 just to prove a point about the average. If you set CloudWatch to display the average with period 1 minute, that's a straight line. 13. Period 5 minutes? 13. Minimum? 13. Maximum? Yes. 

So that's the average over *observations*, not time, in case you expected the sum of all values divided by the time interval. Because that's what "Sum" does. Oh, I'm being confusing now. It's not confusing, really. "Sum" will add up all the values. Choose a time interval that gives you the information you need. Switch to "Samples" for comparison. "Samples" gives you just the observations, no heed to the value 13.

# Muting SDK developers

One more thing. Wouldn't you know it, Amazon uses log4net, so if you use log4net and AWSSDK then you're going to see unintentional log messages from the SDK developers unless you filter them out. Just add the following to your config.

    <logger name="Amazon">
        <level value="OFF" />
    </logger>

# How much?

Custom metrics may put you back. As long as you stay below 10 custom metrics, 10 alarms and 1 million API requests per month, it's free though.

# Why 135?

Keen observers will note that the plot seems to be centered at around 135, not 200 as excepted. What's going on here? Well these requests are going of in sync and they actually take a while, half a second or so. Removing the sleep we'd actually see that we're maxing out.

I admit I didn't realize when I started out. It means that if you're monitoring page loads, for instance, you can't send AWS an observation each time. At moderate loads you're going to max out. A limitation to the usefulness of this project, sure. Not that you'd really want to send of hundreds of http requests a second from your web server, for that matter even text file writes, asynchronous or not.

So if this is your usecase you'll need to aggregate this yourself, as is also recommended by CloudWatch. It's a pain, I know, it's breaking the request cycle, and what's more, beyond the scope of this post. log4net obviously isn't implicitly asynchrounous. Maybe its appenders are and we'd certainly like ours to be. But aggregating is better. How would you accomplish that? Since I'm bent on using this project, I'll probably get around to it soon.

That was it. Download the complete [project](https://dl.dropbox.com/u/1551997/CloudWatchAppender.zip) and play with it. Or fork it from [Github](https://github.com/camitz/CloudWatchAppender).



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
