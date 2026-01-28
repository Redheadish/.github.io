---
layout: page
title: Tags
---

{% assign tags = site.tags | sort %}

{% if tags.size == 0 %}
_No tags yet._
{% endif %}

{% for tag in tags %}
{% assign name = tag[0] %}
{% assign posts = tag[1] %}

## {{ name }} ({{ posts | size }})

{% for post in posts %}
- [{{ post.title }}]({{ post.url }})
{% endfor %}

{% endfor %}
