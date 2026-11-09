---
layout: page
title: 블로그 / 기술문서
icon: fas fa-pen-nib
order: 4
group: category
category_keys: [블로그, 기술문서]
---
{%- assign posts = "" | split: "" -%}
{%- for key in page.category_keys -%}
  {%- if site.categories[key] -%}
    {%- assign posts = posts | concat: site.categories[key] -%}
  {%- endif -%}
{%- endfor -%}
{%- assign posts = posts | uniq | sort: "date" | reverse -%}
{%- if posts.size > 0 -%}
<ul class="content ps-0">
  {%- for post in posts -%}
  <li class="d-flex justify-content-between px-md-3">
    <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
    <span class="dash flex-grow-1"></span>
    <time class="text-muted small text-nowrap">{{ post.date | date: '%Y.%m.%d' }}</time>
  </li>
  {%- endfor -%}
</ul>
{%- else -%}
<p class="text-muted">아직 작성된 글이 없어요.</p>
{%- endif -%}
