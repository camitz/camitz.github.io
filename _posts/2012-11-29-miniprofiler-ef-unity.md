---
date: 2012-11-29 00:35
title: MiniProfiler for Model First EF and Unity
permalink: MiniProfiler-Model-First-EF-Unity
Type: post
tags: [miniprofiler, ef, entity framework, unity container, asp.net mvc]
categories: [Tech, .NET]
---

I know there is good rational behind putting off code optimizations, but I think we've been going overboard with that theme. So it was with relief that my boss told me now is the time to start profiling. Time to install [MiniProfiler](http://www.miniprofiler.com), in other words. Current version is 2.0.2.

I don't know how people can live without it. It's love and kittens for Asp.NET MVC developers, as [Scott H](http://www.hanselman.com/blog/NuGetPackageOfTheWeek9ASPNETMiniProfilerFromStackExchangeRocksYourWorld.aspx) says.

I've been using it for some time on several projects but I haven't installed the Entity Framework till now. That took a bit of head scratching as most of the stuff out there, including MiniProfiler docs, is about Code First setups. Some of my code is getting a bit antiquated, not least my Model First EF 4.2 usage. I just don't know what the stuff on miniprofiler.com means.

## EF MiniProfiler for Model First projects

Nevertheless, [help](http://stackoverflow.com/a/6805478/168390) was available - as countless times before - signed Darin Dimitrov.

    var connectionString = ConfigurationManager
        .ConnectionStrings["MyConnectionString"]
        .ConnectionString;
    var ecsb = new EntityConnectionStringBuilder(connectionString);
    var sqlConn = new SqlConnection(ecsb.ProviderConnectionString);
    var pConn = ProfiledDbConnection.Get(sqlConn, MiniProfiler.Current);
    var context = ObjectContextUtils.CreateObjectContext<CYEntities>(pConn);

## What about Unity Container?

I'm also using Unity Container to inject my object context so if I'm going to put the above anywhere, it's in my container. Previously my container was configured like this.

    IUnityContainer container = new UnityContainer()
    	.RegisterType<CYEntities>(new PerHttpRequestLifetime(), new InjectionConstructor(connectionString));

I need to use a customized piece of code to do the creating for me and I recall this is what is known as a factory. Unity supports this through InjectionFactory.

Hence 

    IUnityContainer container = new UnityContainer()
        .RegisterType<CYEntities>(GetLiftimeManager(lifeTimeManagerType), 
             new InjectionFactory(c=> {
				var ecsb = new EntityConnectionStringBuilder(connectionString);                                                                                                                 var sqlConn = new SqlConnection(ecsb.ProviderConnectionString);                                                                                                                 var pConn = new EFProfiledDbConnection(sqlConn, MiniProfiler.Current);                                                                                                                 return pConn.CreateObjectContext<CoinEntities>(); 
			 }));
	
And it works, too.

