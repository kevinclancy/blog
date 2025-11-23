---
layout: page
title: "Modelling Computer Games"
permalink: /modelling-computer-games/
is_series: true
---

[← Back](/)

# Modelling Computer Games

A series exploring how to model computer games using dynamical systems and lenses.

These posts aim to create mathematical foundations for game-specific programming languages by modeling games as formal mathematical objects. They progress from simple turn-based games to more complex real-time systems with multiple interacting agents.

## Posts in this Series

<ul class="post-list">
{% assign sorted_posts = site.categories.modelling-computer-games | reverse %}
{% for post in sorted_posts %}
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
