---
layout: post
title: Invoking Google Analytics from Jolie
tags:
- tutorials
- microservices
- jolie
---

Do you need to send events to Google Analytics from your Jolie service?
Here is a self-explanatory snippet.

```jolie
interface GoogleAnalyticsIface {
RequestResponse: collect
}

outputPort GoogleAnalytics {
location: "socket://www.google-analytics.com:80/"
protocol: http { format = "x-www-form-urlencoded" }
interfaces: GoogleAnalyticsIface
}

main
{
	collect@GoogleAnalytics( {
		v = 1,
		tid = "UA-XXXXXXXX-Y", // put your own tracking id here
		cid = 555,
		t = "event",
		ec = "Publications",
		ea = "Download",
		el = "mypaper.pdf"
  } )()
}
```

To know what the fields mean, have a look at the official documentation from Google: [https://developers.google.com/analytics/devguides/collection/protocol/v1/parameters](https://developers.google.com/analytics/devguides/collection/protocol/v1/parameters).

I myself find it useful to send events from web servers or other microservices. That snippet is from my own website written in Jolie, to track downloads of my research papers. (Sending these events from the server has the nice consequence that you do not track extra information from the web browser.)

Enjoy!
