---
layout: page
title: Blog
---

{% for post in site.posts %}
## [{{ post.title }}]({{ post.url | relative_url }})

<p class="blog-date">
  {{ post.date | date: "%Y-%m-%d" }}
</p>

{% if post.excerpt %}
<p class="blog-excerpt">
  {{ post.excerpt | strip_html | truncate: 160 }}
</p>
{% endif %}

<hr>
{% endfor %}
