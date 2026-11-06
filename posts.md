---
layout: page
title: Posts
permalink: /posts/
---

<div class="posts-page">
{% assign categories = site.categories | sort %}
{% for category in categories %}
<div class="category-section" id="{{ category[0] }}">
  <h2>{{ category[0] }}</h2>
  <div class="all-posts">
  {% for post in category[1] %}
  <article class="post-item">
    <a href="{{ post.url | relative_url }}" class="post-thumbnail-link">
      <div class="post-thumbnail">
        {% if post.image %}
          <img src="{{ post.image | relative_url }}" alt="{{ post.title }}">
        {% else %}
          <div class="post-thumbnail-placeholder">{{ category[0] }}</div>
        {% endif %}
      </div>
    </a>
    <div class="post-info">
      <h3 class="post-title">
        <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
      </h3>
      <p class="post-excerpt">{{ post.excerpt | strip_html | truncate: 80 }}</p>
      <div class="post-meta">
        <span class="post-date">
          <svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><rect x="3" y="4" width="18" height="18" rx="2" ry="2"></rect><line x1="16" y1="2" x2="16" y2="6"></line><line x1="8" y1="2" x2="8" y2="6"></line><line x1="3" y1="10" x2="21" y2="10"></line></svg>
          {{ post.date | date: "%Y.%m.%d" }}
        </span>
      </div>
    </div>
  </article>
  {% endfor %}
  </div>
</div>
{% endfor %}
</div>
