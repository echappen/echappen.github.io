---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: archive
author_profile: true
---

<ul>
  {% for post in site.posts %}
    <li>
      <p>
        <a href="{{ post.url }}">{{ post.title }}</a>
        <br/>
        <i style='font-size:0.75rem;'>{{ post.date | date: '%B %d, %Y' }}</i>
      </p>
      <p style='margin-top:0.5rem;'>{{ post.excerpt }}</p>
    </li>
  {% endfor %}
</ul>