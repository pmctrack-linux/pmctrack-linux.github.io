---
layout: page
title: News
permalink: "/news/"
---

‎
<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
      {{ post.excerpt }}
   	  <a href="{{ post.url }}">...</a>
    </li>
  {% endfor %}
</ul>


