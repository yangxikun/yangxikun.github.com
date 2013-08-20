---
layout: page
title: Kun Blog
tagline: 记录学习的点点滴滴
---
{% include JB/setup %}

文章列表:

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>