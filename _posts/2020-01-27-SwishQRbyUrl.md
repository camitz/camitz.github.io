---
date: 2020-01-28 12:19
title: Swish QR-code without the API
Type: post
permalink: SwishQRbyUrl
tags: [swish, QR, image, src, img, aws, cloudformation, aws api gateway, aws cloundfront]
categories: [Tech]
---

Can you generate a Swish QR code by placing a url in an image tag, like so:

Yes you can, in fact. Here, I'm doing it. 

<img id="qrimg" src="https://dmutq7l4tk6oq.cloudfront.net/?amount=50&message=Great%20post%21" height="150" width="150">

<input id="qrinput" type="text" value="50" onchange="javascript:document.getElementById('qrimg').src='https://dmutq7l4tk6oq.cloudfront.net/?amount='+document.getElementById('qrinput').value+'&message=Thx for the swish qr post'">

The above image is generated on request, with with the message and amount encoded in the url.

By all means, try it! It will send me a tip of 50 kr (you can change it). I do thank you.

All I did was string together two [AWS](https://aws.amazon.com/) services, [API Gateway](https://aws.amazon.com/api-gateway/) and [Cloudfront](https://aws.amazon.com/cloudfront/), with the Swish [API](https://www.swish.nu/developer#qr-codes). 

If you want to make your own, you can just launch your own stack using the my Cloudformation template. You don't even have to read the rest of the post. Click the icon, enter your Swish number (or your cell phone number), after a few minutes you will have be provided with the domain to paste in your markup in place of the above. 

<a href="https://eu-north-1.console.aws.amazon.com/cloudformation/home?region=eu-north-1#/stacks/create/review
   ?templateURL=https://cf-templates-15y7bciwoyxk8-eu-west-1.s3-eu-west-1.amazonaws.com/SwishQRbyUrl_Distr_0.12
   &stackName=SwishQRbyUrl
"><img src="/assets/images/cloudformation-launch-stack.png" id="qrimg"></a>


You don't even need Cloudfront. That provides caching for the image and I can't really think of any situations where this is not overkill.

A small warning, this may cost you money. Under normal load you won't even hit the free tier limit. Also it's open to the public. Someone could misuse the image and send you lot's of money as well.

If you need help setting this up with you particular requirements, your amount to be editable for example, send me an email.

![](/assets/images/ss_qr_quicklaunch.png "Enter your swish number as an input to the stacks. Change the name if you want.")

![](/assets/images/ss_qr_outputs.png "The output parameter SwishQRDistributionURL is what you will paste in your html code.")

## About the implementation

Swish provides and excellent completely open API for generating QR codes. It required a POST with a JSON payload. You could easily interface it with either backend or frontend code. If you don't have a dedicated Swish number, you'd have to expose your cell phone number, though. (Don't do that.)

You can easily put up a tiny proxy server to customize your needs and your whole organization easily access the QR with imposed limitations, say a closed set of amounts that also the user cannot change or the title. If fixed payee number is a given, otherwise you may find funds being diverted. (It will be the same guy who siphons your latte milk.)

AWS API Gateway, using the HTTP integration type, is great for this exact purpose though. There are alot of considerations working with API Gateway, but basically you can design your interface using the web interface, map it to a model and launch it.

In this case there is just a single GET resource taking two parameters from the query string. We map it to a payload that swish recognizes, like so:

	#set($inputRoot = $input.path('$'))
	{
	  "format": "jpg",
	  "message": {
	    "value": "$input.params('message')",
	    "editable": false
	  },
	  "amount": {
	    "value": $input.params('amount'),
	    "editable": false
	  },
	  "payee": {
	    "value": "1235555555",
	    "editable": false
	  },
	  "size": 300
	}}

The Swish model support more stuff type of image, size, locking of different parameters and more. Above I've halved the image size to 150 pixels directly in the markup. This works well even though Swish API only supports a minimum size of 300.

One thing that is a little out of the ordinary is the binary response. Gateway does support it, but it was actually not part of the original design. You need to explicitly enable support for binary media types image/jpeg and image/webp. It's hidden away in Settings and without it, it will not work properly.

After the basics are set up, you cna configure things like authentication or other restrictions, custom domain name or encryption.

### Why cloudfront?

If you're getting hit by thousands of QR code requests a second you're either being swished alot of money or you're under attack. I don't know how the Swish interface is implemented but presumably they're not going to throttle their oppoprtunity for commission too hard.

In any case, API Gateway will let you impose a throttle if you're worried about this sort of thing. It also let's you easily set up a cache. This uses and compute instance in the background which means it will cost you more than it merits probably.

Another way to implement a cache is to leverage AWS Cloudfront. Cloudfront is a CDN network and since we now have a straight forward url, we can cache it based on the query string. The QR code, given message, amount and other paramaters you desire, will never change so time-to-live can be set infinitely.

Like I said, it's complete overkill but it's a fun excercise. In my case, all I did was set up a distribution mirroring my API, caching on amount and message using the lower of the available price classes. 

Actually I just thought of a use case. Say you have an intermediary Lambda, that reformats the image, changes the size or color or whatever. You might want to cache that.

### The Cloudformation template

I've never used Cloudformation before so that was fun. You download my templates and configure them for your needs. I guess if you're asking this question, then you already know how to find them.

But in short, there are two of them, on for the API Gateway and one for Cloudfront, the former nested within the latter. Both takes the payee number as a parameter and also takes advantage of outputing return values. So the ID of the API is used to construct the url used to invoke the API, which in turn is passed to the parent stack to be used in the HTTP integration. The output of the Cloudfront stack is its domain name, provided by the service.

Enjoy!