---
title: "Tags"
permalink: /tags/
layout: single
author_profile: true
toc: true
toc_label: "Tags"
toc_sticky: true
---

{% assign tags_sorted = site.tags | sort %}

{% for tag in tags_sorted %}
  {% assign tag_name = tag | first %}
  {% assign tag_posts = tag | last %}

## {{ tag_name }}

  <ul>
  {% for post in tag_posts %}
    <li>
      <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
      — <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%Y-%m-%d" }}</time>
    </li>
  {% endfor %}
  </ul>
{% endfor %}
