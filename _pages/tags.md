---
title: "Tags"
permalink: /tags/
layout: page
---

{%- comment -%}
  Build an array of tag names sorted case-insensitively. We use sort_natural
  on the names array (not on site.tags which is a hash).
{%- endcomment -%}
{%- assign tag_names = "" | split: "" -%}
{%- for tag in site.tags -%}
  {%- assign tag_names = tag_names | push: tag[0] -%}
{%- endfor -%}
{%- assign tag_names = tag_names | sort_natural -%}

<p class="tags-intro">{{ site.tags.size }} tags across {{ site.posts.size }} articles. Jump to a letter:</p>

<nav class="tags-alphabet" aria-label="Jump to letter">
{%- assign letters = "A,B,C,D,E,F,G,H,I,J,K,L,M,N,O,P,Q,R,S,T,U,V,W,X,Y,Z" | split: "," -%}
{%- for letter in letters -%}
  {%- assign letter_lower = letter | downcase -%}
  {%- assign found = false -%}
  {%- for name in tag_names -%}
    {%- assign first_char = name | slice: 0 | downcase -%}
    {%- if first_char == letter_lower -%}{%- assign found = true -%}{%- break -%}{%- endif -%}
  {%- endfor -%}
  {%- if found -%}
    <a href="#letter-{{ letter_lower }}" class="tags-letter">{{ letter }}</a>
  {%- else -%}
    <span class="tags-letter tags-letter-empty" aria-hidden="true">{{ letter }}</span>
  {%- endif -%}
{%- endfor -%}
</nav>

{%- assign current_letter = "" -%}
{%- for name in tag_names -%}
  {%- assign first_char = name | slice: 0 | downcase -%}
  {%- if first_char != current_letter -%}
    {%- assign current_letter = first_char -%}
<h2 id="letter-{{ current_letter }}" class="tags-letter-heading">{{ current_letter | upcase }}</h2>
  {%- endif -%}

<section class="tag-section" id="tag-{{ name | slugify }}">
<h3 class="tag-heading">{{ name }} <span class="tag-count">({{ site.tags[name].size }})</span></h3>
<ul class="tag-posts">
{%- for post in site.tags[name] -%}
<li><a href="{{ post.url | relative_url }}">{{ post.title }}</a> <time class="tag-post-date" datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%Y-%m-%d" }}</time></li>
{%- endfor -%}
</ul>
</section>
{%- endfor -%}
