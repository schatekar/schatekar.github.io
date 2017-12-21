---
layout: post
title: A good API does not make assumptions about their consumers
excerpt: A lot of times we build an API to meet a specific need, publish those APIs and then expect the consumers to use the API for that specific purpose. This may work when both API producers and consumers are the same team. But the moment API consumption goes outside of the team, it becomes difficult to dictate how the consumers must use the API. It is best not to make any assumptions around that. 
published: false
---

Recently we found out that an API that was built to help deliver a front-end feature was being used by another team for reporting. Initially, this was not a problem, but as the size of the database grew, we started seeing the impact it had on performance and throughput of this API. How do you deal with a situation like this? Definitely not by asking the other team to stop using this API to build reports. Who knows, tomorrow it will be some other team doing some other thing with this API. The answer is in building a level of protection in your API such that any use of this API will not result in unwanted performance or throughput degradation. The following simple feature additions will give you that protection. 

## #1 - Limit the number of results in a response

An API that returns unbounded result set affects the performance and throughput in more than one way

1. You 
When your API returns an unbounded result set or allows the consumer to request unbounded or a large number of results then you are putting yourself at the risk of letting the API consumer inadvertantly degrade the performance of your API. As a rule of thumb, you should never let your the consumers of your API request an unbounded result set. You can request one or more parameters to filter the results and make those parameters mandatory. Even after this, your API may still return a large number of results. You can limit this by implementing a pagination feature which is always turned on by default. 



## #2 - Limit the response payload


## #3 - Throttle API calls 

## #4 - Prevent spikes in API calls