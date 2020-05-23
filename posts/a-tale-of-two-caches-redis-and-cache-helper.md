---
title: "A Tale of Two Caches: Redis and the cache helper"
description: A note on how to add Redis and the cache helper to a .NET core Mvc app.
date: 2020-05-19T22:06:59+08:00
draft: true
tags: [".NET Core", "cache", "redis"]
thumbnail: https://images.unsplash.com/photo-1569235186275-626cb53b83ce?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=1052&q=80
---

## TODO: Redis Cache

- Why Redis
- StackExchange.Redis (and Core Extension)
- app settings
- build a service
- use it in Controller
- reference

## TODO: <cache> helper in view file

- overview of tag helper in .NET Core 3.1
- cache helper
- apply to razor view : `vary-by-*` directive and `expire-*`
- how to test
- bonus: no helper function anymore: use @Html.PartialAsync() instead
- reference

## Upcoming: building a neat .NET Core Mvc app