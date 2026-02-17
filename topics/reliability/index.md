---
layout: page
title: Blog
---

{% for post in site.posts %}

<article class="blog-item">

  <h2>
    <a href="{{ post.url }}">{{ post.title }}</a>
  </h2>

  <div class="blog-meta">
    {{ post.date | date: "%B %d, %Y" }}
  </div>

  <div class="blog-excerpt">
    {{ post.excerpt }}
  </div>

</article>

<hr>

{% endfor %}

