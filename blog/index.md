---
layout: page
title: Blog
---
{% for post in site.posts %}
### [{{ post.title}}]({{post.url}})

<span style="opacity:0.7;">
  {{post.categories| join: ", "}}
</span>

{% endfor %}
