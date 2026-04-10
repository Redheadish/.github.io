---
layout: page
title: ""
---

<div class="hero">
  <h1>RedHead-ish Tech Blog</h1>
  <p class="hero-subtitle">
    Technical notes on reliability, networking, virtualization, and infrastructure.
  </p>
</div>

<section class="home-section">
  <h2>Latest posts</h2>

  {% for post in site.posts limit:3 %}
    <article class="home-post-card">
      <h3><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h3>
      <p class="blog-date">{{ post.date | date: "%Y-%m-%d" }}</p>
    </article>
  {% endfor %}
</section>

<section class="home-section">
  <p>
    A growing technical notebook for structured notes, experiments, and public engineering writing.
  </p>
</section>
