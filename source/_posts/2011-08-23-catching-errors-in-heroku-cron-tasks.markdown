---
layout: post
title: "Catching Errors in Heroku Cron Tasks"
date: 2011-08-23 21:10
comments: true
categories: 
author: Mauricio Gomes
---

Once we started integrating [RESTful Metrics](http://www.restfulmetrics.com) into our own applications, we quickly ran into some problems with the way we were using the task to invoke other Rake tasks.

Our `cron.rake` looked something like this:

``` ruby
	desc "Heroku Cron Task"
	task :cron => :environment do
      Rake::Task["accounts:bill"].invoke
      Rake::Task["accounts:expire"].invoke
	end
```

As we needed our cron to do more, we just invoked more tasks from within the cron task.

The problem with this approach is that once you start stacking code in your cron task, there is a chance that one of your tasks in the stack will raise an exception and prevent the remaining tasks from being invoked. Worse, unless you check your cron logs frequently, it's likely that your developers aren't aware that these tasks are failing.

In our case, we had created some tasks near the end of the cron stack that sent some data points to RESTful Metrics for aggregation. On the days that our cron script encountered an exception, metrics weren't collected.

To fix this, we've started to wrap each individual task invocation around a begin/rescue/end block and we send the exceptions to [Airbrake](http://airbrakeapp.com). This means that errors generate emails to developers and an error in one task doesn't prevent the remaining tasks from running. The above cron script was re-written to:

``` ruby
	desc "Heroku Cron Task"
	task :cron => :environment do
	   begin
         Rake::Task["accounts:bill"].invoke
	   rescue
	      HoptoadNotifier.notify($!)
	   end
	
	   begin
         Rake::Task["accounts:expire"].invoke
	   rescue
	      HoptoadNotifier.notify($!)
	   end
	end
```

This is a little verbose, but it does the trick for now. Ideally there would be some method that automatically wraps each task invocation in the cron task. This would ensure developers would automatically be protected and the cron task wouldn't become bloated.

What do you guys do?

		