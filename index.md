---
title: "This is a sample page"

---

# Table of contents

This is everything.

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
      {{ post.excerpt }}
      In categories: {{ post.categories }}
      Tags: {{ post.tags }}
    </li>
  {% endfor %}
</ul>