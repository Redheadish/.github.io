---
layout: page
title: Home
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
  <h2>Topics</h2>
  <div class="topic-links">
    <a href="{{ '/topics/reliability/' | relative_url }}">Reliability</a>
    <a href="{{ '/topics/' | relative_url }}">All topics</a>
    <a href="{{ '/tags/' | relative_url }}">Tags</a>
  </div>
</section>

<section class="home-section">
  <h2>About this blog</h2>
  <p>
    This site is a growing technical notebook used for publishing structured notes,
    testing ideas, and building a public engineering portfolio.
  </p>
</section>
