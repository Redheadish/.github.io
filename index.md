---
layout: page
title: ""
---

<section class="hero">
  <p class="eyebrow">Technical notebook</p>
  <h1>RedHead-ish Tech Blog</h1>
  <p class="hero-subtitle">
    Notes on reliability, networking, virtualization, and infrastructure —
    written as a growing public engineering notebook.
  </p>
</section>

<section class="home-grid">
  <div class="home-main">
    <h2>Latest posts</h2>

    {% for post in site.posts limit:3 %}
      <article class="post-card">
        <p class="post-card-date">{{ post.date | date: "%Y-%m-%d" }}</p>
        <h3><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h3>
        {% if post.excerpt %}
          <p class="post-card-excerpt">{{ post.excerpt | strip_html | truncate: 150 }}</p>
        {% endif %}
      </article>
    {% endfor %}
  </div>

  <aside class="home-side">
    <div class="side-card">
      <h2>About</h2>
      <p>
        A curated technical blog focused on systems thinking,
        reliability, and infrastructure practice.
      </p>
    </div>

    <div class="side-card">
      <h2>Explore</h2>
      <p><a href="{{ '/blog/' | relative_url }}">All posts</a></p>
      <p><a href="{{ '/topics/' | relative_url }}">Topics</a></p>
      <p><a href="{{ '/tags/' | relative_url }}">Tags</a></p>
    </div>
  </aside>
</section>
