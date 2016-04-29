---
layout: post
title:  "Scaleio: points of authentication"
date:   2016-04-27 19:25:46 -0400
categories: scaleio authentication
---

The points of authentication in a ScaleIO system may not be obvious.  



I was talking to a user today who was having trouble mostly due to a lack of familiarity with authentication models and the fact that ScaleIO in many ways is unlike traditional enterprise software and more like the families of technologies born of rackscale datacenters.

For example, ScaleIO has a gateway.  If you follow the rules, this is the first entity you bring up in a ScaleIO environment.  Once the gateway is up, you can tell installation manager how you want to craft the environment then sit back and watch the magic happen.  Because the gateway exists
