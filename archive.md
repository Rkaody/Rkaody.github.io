---
layout: page
title: Archive
permalink: /archive/
---

{% assign postsByYear = site.posts | group_by_exp: "post", "post.date | date: '%Y'" %}
{% for year in postsByYear %}
<div class="archive-year">
  <h2 class="archive-year-heading">{{ year.name }}</h2>
  <div class="archive-list">
    {% for post in year.items %}
    <div class="archive-item">
      <span class="archive-date">{{ post.date | date: "%m.%d" }}</span>
      <a href="{{ post.url | relative_url }}" class="archive-link">{{ post.title }}</a>
      {% if post.categories[0] %}<span class="archive-category">{{ post.categories[0] }}</span>{% endif %}
    </div>
    {% endfor %}
  </div>
</div>
{% endfor %}
