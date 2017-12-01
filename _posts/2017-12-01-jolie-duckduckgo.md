---
layout: post
title: Instant web search results from DuckDuckGo in Jolie
tags:
- tutorials
- microservices
- jolie
---

[DuckDuckGo](https://duckduckgo.com/) has an API for instant search results.
It's really easy to invoke and it can reply with JSON or XML data (here I use XML).

Here's a snippet that uses it.

```jolie
include "console.iol"

outputPort DuckDuckGo {
Location: "socket://api.duckduckgo.com:443/"
Protocol: https {
  .osc.search.alias = "?q=%{q}&format=xml&t=jolie_example";
  .osc.search.method = "get"
}
RequestResponse: search // I feel lazy, so I'm using dynamic typing
}

main
{
  // Get the results
  search@DuckDuckGo( { .q = args[0] } )( response );

  // Print results
  println@Console( "# Found " + #response.Results.Result + " result(s)" )();
  for( result in response.Results.Result ) {
    println@Console( "\n" + result.Text + "\n" + result.FirstURL )()
  };
  println@Console()();

  // Print related topics
  println@Console( "# Found " + #response.RelatedTopics.RelatedTopic + " related topic(s)" )();
  for( topic in response.RelatedTopics.RelatedTopic ) {
    println@Console( "\n" + topic.Text + "\n" + topic.FirstURL )()
  }
}
```

Just save this as a file, say `search.ol`, and then you can use it to fetch some results. For example, try `jolie search.ol DuckDuckGo` or `jolie search.ol "Jolie Programming Language"`.

A note about the rules from DuckDuckGo at the time of this writing.
If you use this in some project, remember to update the `t` parameter in the alias of operation `search` from `jolie_example` to something that describes your own project.
