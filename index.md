---
layout: page
title: ""
---

# RedHead-ish Tech Blog

A personal technical blog on reliability, networking, virtualization, and infrastructure.

## Latest posts

{% for post in site.posts limit:3 %}
<div class="home-post-card">
  <h3><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h3>
  <p class="blog-date">{{ post.date | date: "%Y-%m-%d" }}</p>
</div>
{% endfor %}
