---
layout: page
title: Topics
---

{% assign cats = site.categories | sort %}

{% if cats.size == 0 %}
_No topics yet. Categories will appear automatically as posts are added._
{% endif %}

{% for cat in cats %}
{% assign name = cat[0] %}
{% assign posts = cat[1] %}

## {{ name }} ({{ posts | size }})

{% for post in posts %}
- [{{ post.title }}]({{ post.url }})
{% endfor %}

{% endfor %}
