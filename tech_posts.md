---
layout: archive
title:  "Tech"
permalink: /posts/tech
---

{% for post in site.categories.tech %}
  <li><a href="{{ post.url }}">{{ post.title }}</a></li>
{% endfor %}