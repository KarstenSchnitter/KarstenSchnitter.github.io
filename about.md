---
layout: page
title: About
permalink: /about/
---

Software Engineering and Architecture is a private blog on cloud computing and
distributed systems. It provides insights into technical challenges and
solutions, that occur on developing and running a platform-as-a-service on
public and private cloud infrastructure.

## Authors

{% for author in site.authors %}
### {{ author.name }}

**Organization:** {{ author.organization }}  
**Position:** {{ author.position }}  

{{ author.content }}
{% endfor %}
