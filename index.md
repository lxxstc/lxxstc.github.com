---
layout: page
title: 在这聊聊
tagline: Supporting tagline
---
{% include JB/setup %}

{% for post in site.posts limit:2 %}
<div class="post">
  <div class="top">
    <time datetime="{{ post.date | xmlschema }}">{{ post.date | date: "%d %b" }}</time>
    <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
  </div>
  <div class="content">
    {{ post.content}}
  </div>
  <div class="bottom">
    <span>{{post.date | date: "%A %D"}}</span>
  </div>
</div>
{% endfor %}


## Posts

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>

## Jekyll Help

Read [Jekyll Quick Start](http://jekyllbootstrap.com/usage/jekyll-quick-start.html)

Complete usage and documentation available at: [Jekyll Bootstrap](http://jekyllbootstrap.com)
