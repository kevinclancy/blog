---
layout: page
title: "Understanding Module Systems"
permalink: /understanding-module-systems/
is_series: true
---

[← Back](/)

# Understanding Module Systems

This is a series of notes to myself about module systems, undertaken as an attempt to understand them better.

<ul class="post-list">
{% assign module_posts = site.categories.understanding-module-systems | reverse %}
{% for post in module_posts %}
  <li>
    <span class="post-meta">{{ post.date | date: "%b %-d, %Y" }}</span>
    <h3>
      <a class="post-link" href="{{ post.url | relative_url }}">
        {{ post.title | escape }}
      </a>
    </h3>
  </li>
{% endfor %}
</ul>
