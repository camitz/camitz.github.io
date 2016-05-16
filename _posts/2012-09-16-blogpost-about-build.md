---
date: 2012-09-16 20:26
title: Build your dojo/ASP.NET MVC application
Type: post
tags: dojo, build, asp.net-mvc
---

Building your dojo application is a poorly documented pain, in my view. It took me two weeks to get from A to B, not full time, but there was that delay. The only relief comes from Colin Snover at Site Pen who has provided a [boilerplate](https://github.com/csnover/dojo-boilerplate) for us to start from. Its design is for a stand alone dojo application but it's the unavoidable starting point for any project.

I don't have to explain the gains of doing a build, though. That's been done. Suffice to say it's benefial to the point of being crucial.

I am perhaps in a unique situation. Certainly I've realized that I am of a fairly small number in Sweden using dojo. Using dojo in conjunction with ASP.NET probably places me in a select group globally. [Vote](http://aspnet.uservoice.com/forums/41201-asp-net-mvc/suggestions/3132415-add-support-for-dojo-toolkit) on ASP.NET MVC suggestions to contest this view.

## Build it

This is a blueprint of making an dojo build for use in a ASP.NET MVC web environment. The goal is to get all your and dojo's js into a single file, dojo.js. You can split it up later if you want.Make sure to tell me if you find any errors or omissions.

To summarize, even with the boilerplate, it's still painful. Incidentally, derived phrases such "painstakingly", "taking pains to..." all apply. It's a slow process involving small incremental changes, then building, then testing. Use your source control overdiligently. Be smart. Be an engineer.

Everything is centered around and error prone at

1. The dojo javascript include and its dojo-config. 
2. The build script build.bat
3. main.js
4. run.js
5. app.profile.js (or coin.profile.js as I will later rename it to)

Add them all to your source control when appropriate.

## ASP.NET MVC + dojo?

They're not in love, up in a tree, k-i-s-s-i-n-g... What does it mean to use the two in conjunction? Well, for me it means:

1. Lot's of dojo framework code in odd places, both page specific code and application global code.
2. Lot's if dijits on my .cshtml-pages. (razor? yes!) Some are declared in html, some programmatically.
3. Lot's of my own custom widjets, similarly declared in html or programmatically.

At some point in the past I migrated to AMD which was also a pain, but not so much. Mostly it felt very good. [This](http://dojotoolkit.org/reference-guide/1.8/releasenotes/migration-17.html) instruction was invaluable for my widjet. 

Still, for the rest of the application... pain. You need to have complete control over widjet parsing and use lots of ready() and !domready. But in the end, you're also given more control as a result.

If you don't use AMD then I don't know if this blog post will work for you.

## Starting off with the boilerplate

1. Download (or clone) the [boilerplate](https://github.com/csnover/dojo-boilerplate/commit/c952e71a7cbf279319ea717bd41b89a26b735f0a) into a new directory. This is the 1.7.2 version. Lots of other versions are on there as well but I don't know how well this tutorial will apply. At one point I will upgrade and update this post.

2. Did I say I was on windows? It's implicit in .NET perhaps. Never mind, the build script needs modifying and we call the bat-files not sh. Let's use the provided <code>build.sh</code> and rework it. You have java installed, right? This is what mine looks like.


        rem Base directory for this entire project
        set BASEDIR=c:/www/dojo-boiler
        set BASEDIRw=c:\www\dojo-boiler

        rem Source directory for unbuilt code
        set SRCDIR=%BASEDIR%/src
        set SRCDIRw=%BASEDIRw%\src

        rem Directory containing dojo build utilities
        set TOOLSDIRw=%SRCDIRw%\util\buildscripts

        rem Destination directory for built code
        set DISTDIR=%BASEDIR%/dist
        set DISTDIRw=%BASEDIRw%\dist

        rem Module ID of the main application package loader configuration
        set LOADERMID=app/run

        rem Main application package loader configuration
        set LOADERCONF=%SRCDIR%/%LOADERMID%.js

        rem Main application package build configuration
        set PROFILE=%SRCDIR%/app/app.profile.js


        echo Building application with %PROFILE% to %DISTDIR%.

        echo -n "Cleaning old files..."
        del %DISTDIRw% /s /q
        echo " Done"

        cd %TOOLSDIRw%

        java -Xms256m -Xmx256m  -cp ../shrinksafe/js.jar;../closureCompiler/compiler.jar;../shrinksafe/shrinksafe.jar org.mozilla.javascript.tools.shell.Main  ../../dojo/dojo.js baseUrl=../../dojo load=build --require %LOADERCONF% --profile %PROFILE% --releaseDir %DISTDIR%

        cd %BASEDIR%

         dir dist\dojo\dojo.js

        echo "Build complete"

    Pain! Yes, I believe I said so. Most variables have forward slash and backslash variants. Go ahead and tell me about the better way to do this.

    The first lines should reflect your absolute basedir. Build it. It should work. Surf to <code>index.html</code>. It should work.

3. I've made one functional addition above. It's the <code>dir</code>-thing. If the build does not work, <code>dojo.js</code> will be of size zero so this is the check that things are going in the right direction. 

	What could go wrong? Well, I found it very difficult with trailing commas here and there in my javascript. This is wrong but there are no browsers that warn me about it and no visual studio either. What happens is simply a <code>dojo.js</code> of size zero. That was until Ken, who's a star of the dojo-interest mailing list gave me this:

        // Reduce logging level to WARNING in Closure
        if (typeof Packages !== 'undefined' &&
            Packages.com.google.javascript.jscomp.Compiler) {
                  //Packages.com.google.javascript.jscomp.Compiler.setLoggingLevel(Packages.java.util.logging.Level.WARNING);

                  Packages.com.google.javascript.jscomp.Compiler.setLoggingLevel(Packages.java.util.logging.Level.SEVERE);
        }
    
    Trailing commas and anything else that closure can't get its head around shows up as errors. Increase your command window buffer size.

    Also check the <code>build-report</code>. It's more of an interesting read than anything else. Some errors about i8n and plugins appear. These are known issues for dojo 1.7.

4. At this point I upgraded to 1.7.3. If you want, download and replace all files. Build and test.

    ## Preparations in your javascript

5. Make sure everything is prepared, nice, good and correct. All must be encoded in ANSI or UTF8, preferably the latter. Not UTF8+. 

    You're using AMD, you have relative paths for modules, you have no trailing commas in you dependency lists like I did. Make sure all your widjets are in a single subdirectory to Scripts. If this subdirectory is <code>app</code>, lucky you. Mine was <code>coin</code>.

6. Make sure there are no javascript errors on your site. Surf around with a developer panel like Firebug open. Use different browsers. Diligence.

7. Copy dojo from the boilerplate locally, dojo, dijit, dojox and utils, the works. Set to load <code>dojo.js</code> from this place. I was using Google's CDN prior to this, even for my local development. Maybe you have your dojo locally but in a directory way over there? That's not what I mean. I mean *in* you development directory, *in* your ~/Script directory. You don't have to include it in the VS project or in you source control, though. I've given this thought but I had so many problems with the basedir and relative paths to scripts and locale bundles that... well, I'm not going there.

8. Try your app. Run UI tests if you have them. Surf to a page that has dijits declared in markup if you have them. This will convince you that the parser is running when it should. Keep an eye on Firebug for js and network errors.

    ## Boilerplate goes in there

9. Copy the boilerplate locally. So that's the app directory, <code>index.html</code> and the build script if I'm not mistaken. Adjust your build script. Build and surf to <code>index.html</code>. It should work. That means all the directories know each other.

    ## Bring it together

10. I decided to parse my pages in <code>main.js</code>. It's a nice place to start your application. One line is:

        define([ 'dojo/has', 'require',  'dojo/ready', 'dojo/parser' ], function (has, require, ready, parser) {

    Some others are:

          ready(function() {
              parser.parse();
          });

    The complete file is lost to me now. You'll figure it out.

11. Build. Well, you haven't changed anything really, so it will still build. Right? That's what I'm talking about. You can't build and test enough. Things will go wrong and you just can't see it.

12. No, my app is called <code>coin</code> not <code>app</code>. So everything in the <code>app</code> directory goes in the <code>coin</code> directory or whatever your one is called. Copy, don't move. Leave <code>app</code> in place for now. Update the build script using search and replace. The comments will have words like *coinlication*. Why should that bother you? Do the same for what's now called <code>coin.profile.js</code> and <code>run.js</code>. <code>baseUrl</code> is fine the way it is i.e. ''.

13. The contents of <code>index.html</code> should go into your layout definition. This is the starting point for the dojo part of the application. I put this in a separate file, <code>dojo.cshtml</code>, that I include with 
     
        @Html.Partial("Partials/Dojo")

    The snippet has defer on each of the two script tags. Remove the first.

    Build. The web site should now work again, augmented with the little Hello worlds dialog from the boilerplate. Importantly, we're now loading <code>run.js</code> (from the <code>dist</code> directory, of course). <code>run.js</code> is used by the builder and also by the site, so it's a very important file to get right.

    ## One by one

14. All the dijits and custom widjets that were declared in HTML: now is when I start including them in the build. Formerly they hung loose on a layout page or a specific cshtml. May have had some idea about this or that widjet not being required unless someone ventures into this or that page. That sort of thinking is great but now is not the time for it.

    I'm bringing them into the bottom of my run.js, the dependency list in square brackets. You can also put them in your profile if it makes you feel better.

    One by one folks! Start with factory made dijits. Include it in <code>run.js</code>. Build. Test. Proceed with your custom widjets. 
    
    I can't stress how totally worth your time this is, even though the build may take minutes. Read up on some blogs or whatever during the build. Lots to learn.

15. Now it's time to start loading the site from <code>dist/dojo</code>. Change it in <code>dojo.cshtml</code>.

    Start with uncompressed (<code>dojo.js.uncompressed.js</code>) version so you can debug it if there is a problem in your own modules. 

    ## Locale bundles and other resources

    At this point, let me pause for a minute. Perhaps you are having problems with locale? Loading bundles in <code>cldr</code>, <code>nls</code> and what have you? Other resources? <code>blank.gif</code>? 

    I had painful problems. It seemed like the client would look in arbitrary locations for them. If I'd navigate to <code>Controller/Action</code>, I'd see network errors for <code>&Controller/Action/dist/dojo/resources/blank.gif</code>. Sometimes it would work in one browser and not the other. Sometimes in the uncompressed <code>dojo.js</code> while it worked fine in the compressed one. I experimented with <code>baseUrl</code>s and everything I could think of. In the end I just hacked it. I put this in <code>Global.asax.cs</code>.

        protected void Application_BeginRequest(object sender, EventArgs e)
        {
            string path = Request.Url.PathAndQuery;

            if (!path.Contains("dist/"))
            {
                if (_regex.IsMatch(path))
                {
                    var match = _regex.Match(path);
                    Context.RewritePath("/dist/" + match.Groups[1].Value + 
						match.Groups[2].Value + match.Groups[3].Value);
                }
            }
        }

    What that does is make sure the if dist is in the requested url\s path then rewrite the url to make sure that dist is the root. Nothing else make's any sense.

    In the end I also modified the dojo source code for locale failures in ValidationTextBox and CurrencyTextBox which worked in everything except Firefox. Some kind of locale thing failing to load. That will strike back, I know, but later and perhaps no after I upgrade to 1.8.

16. Take a look in the network listing of whatever developer panel you're using. You may see some files, although loaded from the <code>dist</code> directory, they're still being loaded separately. They've not been included in the build for some reason. You'll need to include each one specifically in your layer, in your <code>xxx.profile.js</code>. Don't change anything. Just add to it.

    Take a break when all that remains is <code>dojo.js</code> and is <code>dojo_sv.js</code> or whatever locale you're in. It's meant to be that way. 

    At some point I also had a couple of <code>common.js</code> in there. I'm not sure why they're gone now. If you have them, just leave them, perhaps you'll be lucky, too. Not worth fretting over, anyway.

17. Now start removing dependencies on the <code>Scripts</code> directory. Use your network panel and file search.

    I had some files that were AMD but I had put them in directly in Scripts. No real reason. Not really part of any widjet or app per say, not really pure "script" either. Put it in the <code>coin</code> subdirectory and reference it from <code>run.js</code>. Feel good about that.

    I also had a collection of odd functions in <code>Helpers.js</code>. Some are now included in the one page that happened be using them. Others I put directly in my main layout file. That left me with two pure "script" files that were loaded separately. Someday I'll make them AMD, good and proper.

    Build. Test.

18. Remove the part in <code>run.js</code> which displays the <code>Dialog</code>. Remove <code>Dialog.js</code> and any references. 

	Build. Test.

That's all there is to it!

ZQCGE4H2JF35 

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
