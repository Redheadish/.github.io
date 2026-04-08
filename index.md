---
layout: page
title: Home
---

# RedHead-ish Tech Blog

Technical notes on systems, reliability, networking, virtualization, and infrastructure.

## Latest posts

{% for post in site.posts limit:3 %}
### [{{ post.title }}]({{ post.url | relative_url }})

<p class="blog-date">
  {{ post.date | date: "%Y-%m-%d" }}
</p>

{% if post.excerpt %}
<p class="blog-excerpt">
  {{ post.excerpt | strip_html | truncate: 140 }}
</p>
{% endif %}

{% endfor %}

## Topics

- [Reliability]({{ '/topics/reliability/' | relative_url }})
- [Topics]({{ '/topics/' | relative_url }})
- [Tags]({{ '/tags/' | relative_url }})

## About this blog

This site is a growing technical notebook.
It is used for publishing structured notes, testing layouts, and building a public engineering portfolio.
