---
layout: archive
title:  "Life"
permalink: /posts/life
---

{% for post in site.categories.life %}
  {% if post.url %}
    <li><a href="{{ post.url }}">{{ post.title }}</a></li>
  {% endif %}
{% endfor %}