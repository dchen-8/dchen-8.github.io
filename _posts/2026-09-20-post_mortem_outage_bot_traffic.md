---
layout: post
title: "Post Mortem - Outage - Bot Traffic"
author: David Chen
email: david@davidc.dev
date: 2026-09-20
categories: document learnings writeup post-mortem outage
---

* TOC
{:toc}

TLDR: Outage happened on September 09, 2026 because of runaway instance spawning from bot + normal traffic. Mitigation is to add bot prevention methods and to add safe start up procedures for Google App Engine (GAE).

## Intro

On September 20, 2026 there was an outage where I could not access my website Chen.Trading. The initial spike started at 8:35AM Pacific Time (Based on server logs), requests went from 0.64/s to peak of 6.1/s within the next 30 mins. No new server instances were spin up during this time, a single server was able to handle all this traffic, the issue started spiralling around 12:40 Pacific Time when the server started sending 500 status code errors. There was a 3.23/s of traffic with a 2.04/s of errors.

I believe that is when I started using my website and started noticing that it was really sluggish. I believe my actions in using my website started the cascading issue. Looking back at the logs, it seems the bots were barely hitting the server enough where it had to spin up new server instances but when it did, it was a max of 4 instances before they started to spin down and go back to a default of 1 active server. Once, I started using my site, there was more of a load of complex queries and usage that was on more data intensive paths that would normally be avoided by APIs such as Watchlist, Profile and Search. 

Once, I started the ball rolling the server seemed to need to create more servers instances but that's when it seems to have gone really wrong. The server instances each came up as bad and would not be able to handle the traffic so the server created more instances to help handle the load. The problem was the connection limitation between Google App Engine and CloudSQL. There is a hard limit of 25 connections for a Postgres (db-f1-micro) instance, max_connections can be modified on the CloudSQL instance but would run the risk of OOM(Out-of-Memory) crash. During this period Google App Engine spawned 30 instances that all were trying to create a connection to CloudSQL but each one failed so it tried to spawn a new instance each time. I did not see a way out of this cascasing issue, more request would be coming through repeat requests and each Google App Engine instance was created and immediate failed without being able to server anything, lastly my CloudSQL instance was probably close to crashing as it was maxed on connections and there was little it could do to stop the hammering of queries. 

I decided at 1:00 PM Pacific to disable Google App Engine and have all servers go down. There was no way to stop the damage as Google App Engine was unresponsive and turning off CloudSQL most likely would not have stopped the issue, it might have just kept spawning GAE instances. Once, I clicked the button it took about 5 minutes for all the traffic to die down and just have nothing happen or show up in the logs.

That was the end of the incident. Recovery and steps for mitigation will be described in the next sections.

There was another flare up at 3:10P PM Pacific where traffic seemed to ballon up again with request of 6.69/s but with errors of 3.76/s. Mitigated the issue with Cloud Flare rate limiting verified bots and eventually blocking the offending bot outright.

## Timeline

* 8:40 AM Pacific Time - Bot Traffic is starting
* 12:40 AM Pacific Time - Incident Starts
  * Chen.Trading - Inundated with request; 3.23/s of Requests; 2.04/s of Errors
  * Google App Engine - Spawns 30 Instances
  * CloudSQL - Max Connections: 25, Total Connections: 29; 64% Utilization
* 1:00 PM Pacific - Google App Engine Disabled
* 1:05 PM Pacific - Traffic is 0
  * Chen.Trading - Website is non-functional; Constantly pinging with only a white screen
  * Google App Engine - Disabled; All traffic is ignored; No servers are up
  * CloudSQL - Connections drop down to normal levels
* 1:10 PM Pacific - Incident should be officially over

## Root Cause

### What Happened

The server for Chen.Trading got caught in a infinite loop where large amount request came in all at once, which overwhelmed the first server, so another one was spun up but as soon as the second server tried to serve data, it errored out and so an third server was spun up. This caused cascading failure for each new server that was created to attempt to take another CloudSQL connection but then eventually ran out of connections so it caused a secondary error.

### Why it happened

This happened because the /_ah/warmup procedure for each Google App Engine was a pass through which did not do any connection checking before marking the server as ready to serve traffic. With small amount of traffic this usually isn't a problem but when resources start to get constrained this warmup procedure becomes very important. 

Had this warmup procedure existed then potentially the worst thing that would have happened is that a second server would have been spun up, checked all the connections are active then started serving user requests while the first server handled the bulk of the bot requests.

This should have been a blip on the traffic.

### Contributing factors

The other contributing factor is that there was no robots.txt to disallow traffic to specific routes so that a bot crawler trying to get as much data as possible should have slowed down.

There was also no type of rate limiting of traffic anywhere. Cloud flare does a great job of dealing with the brunt of the traffic from caching pages. 

There was no max limit to number of Google App Engine instances that could have spawned which could have theoretically been hundreds.

Google App Engine server was not configured to check traffic before passing along requests. Had there been any kind of rate limiting or blocking mechanism, it could have been trivial to stop the bot.

CloudSQL could have been upgraded to a full instance instead of a shared core instance with only a max of 25 connections. This couldn't have stopped the attack but it could have slowed the second round of GAE instance failures.

Alerting could have caught the large amount of traffic and alerted me. No alerts were setup to catch anything.

## Action Items

### Immediate Fixes

Things that were implemented to help resolve this incident.

#### Google App Engine

* Set Max Instances to 10
  * This will mean that only a max of 10 connections will ever spawn to connect to CloudSQL. Which would prevent any kind of instance sprawl.
* Fixed and actually implemented a warmup procedure for Google App Engine
  * By setting a warmup procedure for /_ah/warmup, the server instance will attempt to make connection to service before it send the signal that it is ready to receive traffic. 
* Implemented verified bot blocking through FastAPI Middleware for direct API access
  * If bots try to query API directly, they will be denied but if they are request my website URL and the source and referrer match, they will be allowed through

#### Chen.Trading Website

* Added robots.txt and sitemap.xml
  * Set the allow and disallow paths for bots and AI agents

#### Cloud Flare

* Set Rate Limiting rule for verified bots
  * This worked really well. Any bots that exceeded 1/s would be blocked for 10 seconds.
* Blocked the offending bot's User Agent
  * Determined that 90% of all the issue came from one company's bot, block the bot and all the issues went away

### Preventative Measures

Things that will be put in place to help prevent an issue like this happening again.

#### BigQuery

* Cloud logs are being exported into BigQuery for deep diving into the analytics

#### CloudSQL

* Query Insights was enabled to see any long running queries and frequency of similar queries

#### Cloud Logging / Alerts

* Alerts should be set up to trigger on things like website is not reachable or there are more errors than usual

## Lesson Learned

### What went well

* Cloud Logging and having the whole stack on Google Cloud Platform made it pretty trivial to look back into the logs to see everything that happened. 
    * It was trivial to see what was happening through the dashboard on each service but being able to do anything about it was a different story.
* Google App Engine seemed to handle about 6/s requests pretty well for one server. 
    * Spawning more instances seemed to work well and would have scaled up to as many as needed for large amount of traffic. 
* CloudSQL handled extremely well; It did not crash nor did it seem like it was struggling too much.
    * Queries seemed generally optimized and caching results from GAE seemed to block the brunt of the incident from CloudSQL
* Chen.Trading seemed resilient, in that once the attack was over, it was able to pick right back up with the server being down. 
* Cloud Flare did an immensely impressive job of blocking a large portion of the requests.
    * I believe that if Cloud Flare was not hosting my static files, the amount of traffic would have crippled my site way earlier.
* Redis seemed to have done a fantastic job and just kept serving data without issue even when more servers were being spawned. 
    * Peaked at 15 connections
    * Hmm, that is weird my throughput is 200% at 200 ops per sec

### What could be improved

* Alerting
  * If I had a sense of what was happening earlier, I could have added measure earlier rather than let it get to this point. 
* Google App Engine
  * There is no way to stop an instance other than to disable it. I was looking for some way to pause it but that did not seem to exist and with how many instances were being spawned, I could not manually stop each one.
  * Good and Bad, can't stop it without disabling it.
* CloudSQL
  * I should consider moving to a full core so I can take advantage of the more number of connections to be able to serve more data. 
* Redis
  * I should probably move to the next tier where I can hold 250 MB in memory and have 1000 ops/sec with up to 256 connections. 

Overall it was a harrowing situation but I am glad I was able to experience it, get through it and I believe it was only $1.60 in Google App Engine instance costs. 

## Appendix 

### Google App Engine - Log Errors

* 420 occurrences of:
psycopg.OperationalError: connection failed: connection to server on socket "[CONNECTION]" failed: FATAL: remaining connection slots are reserved for roles with privileges of the "pg_use_reserved_connections" role

* 420 occurrences of: \
 sqlalchemy.exc.OperationalError: (psycopg.OperationalError) connection failed: connection to server on socket "[CONNECTION]" failed: FATAL: remaining connection slots are reserved for roles with privileges of the "pg_use_reserved_connections" role


### Cloud Flare Stats: on September 20, 2026 - 3 PM Pacific
 * 52.11K Requests served by Cloud Flare
 * 14.59K Served by Origin (Chen.Trading)