---
layout: page
title: Tags
permalink: /tags/
---

# Tags

Browse posts by topic:

{% assign tags = site.tags | sort %}
{% for tag in tags %}
  <h3 id="{{ tag[0] | slugify }}">{{ tag[0] }}</h3>
  <ul>
  {% for post in tag[1] %}
    <li><a href="{{ post.url }}">{{ post.title }}</a> — {{ post.date | date: "%Y-%m-%d" }}</li>
  {% endfor %}
  </ul>
{% endfor %}
