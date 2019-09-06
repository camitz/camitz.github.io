---
date: 2015-01-05 16:00
title: EC2 Spot Instance Termination Notification to SNS
Type: post
permalink: EC2SpotStop2SNS
tags: [amazon, AWS, SNS, spot instance, termination, windows, instance meta data]
categories: [Tech, AWS Appender]
---

The AWS team yesterday [announced](https://aws.amazon.com/blogs/aws/new-ec2-spot-instance-termination-notices/?sc_ichannel=em&sc_icountry=global&sc_icampaigntype=launch&sc_icampaign=em_130420040&sc_idetail=em_66267057&ref_=pe_395030_130420040_8) that it would publish notifications of imminent termination of spot instances in the form of a date string accesible via the AWS [instance metadata service](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html), the date being the time of expected termination. Termination of spot is usually due to the spot price exceeding the configured maximum price.

So now the instance can find out that it is being terminated and can take approprate action and that's really great. But I want to know aswell!

And Amazon has really great infrastructure for notifications called [SNS](http://aws.amazon.com/sns/). Service can subsribe to topics and it can send emails and more. Connect the two and I can find out when my spots are terminated.

That's just me, for my personal satisfaction of knowing. But what other people will find even more useful is of course other applications subscribing to these notices, other spot instances for example. Maybe they need to adjust their games in the event of fellow instances being dropped. That reduces the need for a central controller and instances can become more autonomical.

## Introducing [EC2SpotStop2SNS](https://github.com/camitz/EC2SpotStop2SNS)

Nothing fancy. You can run it from the command line or install it as a service. It even has an installer. 

As per the recommendations of Jeff Barr, it will poll the metadata service every 5 seconds. If the termination notice appears it immediately publishes a message to SNS.

If there isn't a topic on SNS it will create one when you run it the first time. Goto AWS management console and configure the topic to throw you an email or text once something is published.

That would have been the end if there wasn't a second need that I've come across before.

## Introducing [MockEC2InstanceMetaData](https://github.com/camitz/MockEC2InstanceMetaData)

How can I debug this locally? Already when I was working an the early versions of log4net [CloudWatchAppender](https://www.google.se/url?sa=t&rct=j&q=&esrc=s&source=web&cd=2&cad=rja&uact=8&ved=0CCwQFjAB&url=http%3A%2F%2Fblog.simpletask.se%2Fpost%2Fbuffering-aggregating-cloudwatch-appender&ei=gmqtVI_AMeSZygPXn4LwBA&usg=AFQjCNHtqHMn2vWFqauv_xnzxoRC8pyJDw&sig2=r_NhyK-8uIvHGe_yLVNX3w&bvm=bv.83134100,d.bGQ) I realized this need. I just winged it back then.

So this is a bit of ad hoc programming. It uses the an Owin stack to listen in on 127.0.0.1:9000 (the port is configurable). Not useful unless you redirect traffic from 169.254.169.254. I think I could figure out how to listen to 169.254.169.254 but decided to save the hassle on something [Fiddler](http://www.telerik.com/download/fiddler) does so well. It works like your windows hosts file except allows for ports as well.

Then the app will answer all your calls to it producing something hopefully similar to what you get from the real metadata service. It worked splendidly for debugging EC2SpotStop2SNS, at the very least.

The data is completely configurable via a json file and what's more, it reads the file everytime you want a response, so you don't have to restart the application when you make a change.

## What's next?

Obviously I couldn't be bothered to configure my json with all the data from a particular instance metadata service. Now all I need is an app the does that for me. Shouldn't be that hard to do. You're welcome to help me out on that one.