---
layout: page
title: blog
permalink: /blog/
description: Occasional notes on explainable AI, evaluation, and research practice.
nav: true
nav_order: 3
---

{% for post in site.posts %}
  <article class="mb-4">
    <h2 class="h4"><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
    <p class="post-meta">{{ post.date | date: '%B %Y' }}</p>
    {% if post.description %}<p>{{ post.description }}</p>{% endif %}
  </article>
{% endfor %}
