+++
title = "awk-wardly automating Heroku Postgres forking for fun and profit"
date = "2023-03-09"
tags = ["heroku", "devops", "programming", "work"]
keywords = ["heroku", "devops", "postgres"]
description = "Because we're too good for pg:copy"
draft = true
+++

At `$DAYJOB`, we do some interesting things.

And I'm not just talking about the problems we solve. While the day-to-day work isn't necessarily very flashy, the logistical challenges and "toss it up now, figure it out properly later" attitude you find in a lot of startups mean I get to work on a lot of interesting side tasks. For instance, todays's topic: how can we copy a database from one application to another to seed it with test data? Specifically, goal here is to maintain a staging environment which mimcs production as closely as possible, not just in code, but in data as well (with customer data purged and/or anonymized, obviously). To that end, we set out to automate this synchronization and anonymization to occur every night, so that our environment would be clean and ready for work the next day.

For some background: `$DAYJOB` has been a long-standing customer with [Heroku](https://heroku.com). If you don't know already, Heroku is a popular platform-as-a-service solution which has had some... stagnation in its development in recent years, to put it lightly. To make a long story short, following an acquisition by Salesforce, I'm told Heroku is effectively operating on a skeleton crew strong enough to "keep the lights on," so to speak. This "stagnation" is going to play a key role in today's story, as well as future posts. 

Back to the original topic. Those of you with some familiarity with Heroku's tooling might be thinking to yourself, "Heroku's CLI has a fancy command that does exactly what it says on the tin for this: [`pg:copy`](https://devcenter.heroku.com/articles/heroku-postgres-backups#direct-database-to-database-copies)." And you'd be right! Heroku does allow you to initiate a direct database-to-database transfer of data, which sounds perfect. In fact, up until recently, this is exactly the command we'd been using. Every evening at around 06:00 UTC, our little application would wake up and begin a sync between prod and staging, just as designed. This had been running fairly reliably since before I even started, barring the occasional hiccup. However, as time progressed and our dataset grew (even through a rebrand and a rearchitecture), we started running up against an increasing amount of odd failure modes, ranging from severely degraded performance from a batch of indices being missed during the copy/restore process to the copy just.. .failing to start at all, with the ever-helpful `Internal Server Error` being kicked back at us. To me, it was becoming clear that this was starting to become less maintainable, and so I began searching for other alternatives.

As it happens, Heroku actually has a couple of ways to copy your databses. In addition to the obvious `pg:copy`, you also have the ability to ["fork" an existing database into a new one](https://devcenter.heroku.com/articles/heroku-postgres-fork#create-using-the-cli). This is done by creating a brand new Heroku Postgres add-on, but passing in some special flags which effectively copies a stored backup of the Postgres data directory, plus any new WAL snapshots, from the source to the destination. The command for doing this looks a little something like this:

```
heroku addons:create heroku-postgresql:standard-0 \
  --app your-destination-app-here \
  --fork your-source-app-here::DATABASE_URL \
  [--fast]
```