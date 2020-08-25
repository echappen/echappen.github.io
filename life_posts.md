---
layout: archive
title:  "Life"
permalink: /posts/life
---

{% for post in site.categories.life %}
  <li><a href="{{ post.url }}">{{ post.title }}</a></li>
{% endfor %}
