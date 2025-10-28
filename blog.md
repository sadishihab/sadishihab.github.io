---
layout: default
title: Blog
permalink: /blog/
---

# My Blog

**Stay updated:** 
[Subscribe via RSS](https://sadishihab.github.io/feed.xml)
<a href="https://sadishihab.github.io/feed.xml">
  <img src="https://upload.wikimedia.org/wikipedia/commons/4/43/Feed-icon.svg" alt="RSS Feed" width="20" style="vertical-align: middle;">
</a>

Here are my latest posts:

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url | relative_url }}">{{ post.title }}</a> — <small>{{ post.date | date: "%b %d, %Y" }}</small>
    </li>
  {% endfor %}
</ul>
