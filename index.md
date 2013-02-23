---
layout: page
title: 在这聊聊
tagline: Supporting tagline
---
{% include JB/setup %}

{% for post in site.posts reversed limit:10 %}
<div class="post">
  <div class="top">
    <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
  </div>
  <div class="bottom">
    <time datetime="{{ post.date | xmlschema }}">{{ post.date | date: "%Y/%m/%d" }}</time>
  </div>
</div>
<hr/>
{% endfor %}


## Jekyll Help

Read [Jekyll Quick Start](http://jekyllbootstrap.com/usage/jekyll-quick-start.html)

Complete usage and documentation available at: [Jekyll Bootstrap](http://jekyllbootstrap.com)
